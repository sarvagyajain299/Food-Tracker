# Food Tracker v2 Reply Formatting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `Food Tracker v2`'s Telegram replies scannable (HTML formatting) and make food-data provenance visible (📋 database vs 🌐 web lookup), and show both today's running total and remaining after a successful log.

**Architecture:** This is an n8n workflow edit, not application code. Two nodes change on the live workflow `YOUR_WORKFLOW_ID` (Food Tracker v2): the `Calorie Logging Agent`'s `systemMessage` (prompt rewrite) and `Telegram: Send Message`'s output expression (stop escaping the agent's own HTML tags). No new nodes, no connection changes, no changes to the `Log Food Entries` sub-workflow.

**Tech Stack:** n8n Workflow SDK (via the connected n8n MCP server — see project `.claude/CLAUDE.md` §0 tool table), `@n8n/n8n-nodes-langchain.agent` system prompt, n8n expression syntax, Telegram Bot API HTML parse mode.

## Global Constraints

- Never guess n8n syntax — `get_sdk_reference` before writing SDK code, `get_node_types` before touching parameters (CLAUDE.md §1.1).
- Snapshot node JSON to `workflow_versions/Food_Tracker_v2/<YYYY-MM-DD_HHmm>_pre-reply-formatting.json` before calling `update_workflow` (CLAUDE.md §-1).
- After `update_workflow`, call `get_workflow_details` and read `connections` before treating the change as done (CLAUDE.md §1.2) — this plan doesn't touch connections, but verify they're unchanged.
- No secrets, no invented credential IDs (CLAUDE.md §1.3) — not applicable here (no credential fields touched), stated for completeness.
- Timezone: prompt already anchors dates via `$now.setZone('Asia/Kolkata')` — preserve this, don't strip it while rewriting the system message.
- Ask before `execute_workflow` — use `test_workflow` with pinned data instead (CLAUDE.md §1.4).
- Do not `publish_workflow` — the workflow is already active; `update_workflow` on an active workflow takes effect without a separate publish step. Do not deactivate/reactivate.
- Append a `.claude/WORKFLOW_LOG.md` entry after the update, per CLAUDE.md §-1.

---

### Task 1: Snapshot current node state (local version control)

**Files:**
- Create: `workflow_versions/Food_Tracker_v2/<YYYY-MM-DD_HHmm>_pre-reply-formatting.json` (timestamp = actual current time when this task runs)

**Interfaces:**
- Consumes: `get_workflow_details` output already fetched for `YOUR_WORKFLOW_ID` (see spec) — specifically the `Calorie Logging Agent` and `Telegram: Send Message` node objects from `nodes[]`.
- Produces: a JSON snapshot file later tasks can diff against or restore from if the update needs rollback.

- [ ] **Step 1: Call `get_workflow_details` for `YOUR_WORKFLOW_ID`**

Confirms current state hasn't changed since the design was written (Telegram Trigger, Calorie Logging Agent, Telegram: Send Message node IDs and params).

- [ ] **Step 2: Write the snapshot file**

Save the full `nodes` array and `connections` object from the response to the path above, e.g.:

```json
{
  "nodes": [ /* full nodes array from get_workflow_details */ ],
  "connections": { /* full connections object */ }
}
```

- [ ] **Step 3: Verify the file was written**

Read the file back and confirm it contains the `Calorie Logging Agent` node with its current (pre-change) `systemMessage`, and the `Telegram: Send Message` node with its current `text` expression (`.replace(/&/g, '&amp;')...`).

---

### Task 2: Look up current SDK/node parameter shapes before editing

**Files:** none (read-only MCP calls)

**Interfaces:**
- Consumes: node type names `@n8n/n8n-nodes-langchain.agent` (v3.1) and `n8n-nodes-base.telegram` (v1.2), already known from Task 1's snapshot.
- Produces: confirmed parameter paths (`parameters.options.systemMessage` for the agent; `parameters.text` for the Telegram node) that Task 3 and Task 4 write to.

- [ ] **Step 1: Call `get_sdk_reference`**

Read the SDK code-authoring reference before constructing any `update_workflow` code payload — required by CLAUDE.md §0/§1.1 regardless of how small the edit is.

- [ ] **Step 2: Call `get_node_types` for `@n8n/n8n-nodes-langchain.agent` and `n8n-nodes-base.telegram`**

Confirm `options.systemMessage` (string, supports `=` expression prefix for `{{ }}` interpolation) and `text` (string, same expression syntax) are still the correct parameter names/paths at the installed node versions (3.1 and 1.2 respectively, per the snapshot). If either version has drifted, adjust field names in Tasks 3–4 accordingly instead of assuming the spec's names are still valid.

---

### Task 3: Rewrite the Calorie Logging Agent system prompt

**Files:**
- Modify (via `update_workflow`, not a local file): `Calorie Logging Agent` node (`id: 8c74117d-c517-4619-9d78-5b98f0ab429b`) in workflow `YOUR_WORKFLOW_ID`.

**Interfaces:**
- Consumes: existing prompt variables already interpolated into the current `systemMessage` — `{{ $now.setZone('Asia/Kolkata').toFormat('cccc, dd LLLL yyyy') }}`, `{{ $json.foodList }}`, `{{ $json.targetLine }}`. These must all be preserved verbatim; this task only changes instructional text, not the data-binding expressions.
- Produces: a `systemMessage` string that later tasks (5, 6) test against via `test_workflow`.

- [ ] **Step 1: Compose the new `systemMessage` string**

Keep every existing section (identity, food database dump, today's target, lookup order, quantities, arithmetic, tone) and modify/add only the CONFIRMATION, AFTER LOGGING, and FORMATTING guidance. Full replacement text:

```
You are a calorie logging assistant. Today is {{ $now.setZone('Asia/Kolkata').toFormat('cccc, dd LLLL yyyy') }}.

Your job is to log what the user eats, accurately and fast.

MY FOOD DATABASE (id|name|serving|calories|protein|carbs|fat):
{{ $json.foodList }}

TODAY'S TARGET:
{{ $json.targetLine }}

LOOKUP ORDER — never skip a step:
1. Match against the food database above. It is already in front of you — there is no tool to search it and you must not ask for one.
2. Only if a food is genuinely absent, call lookup_food_on_web, then add_new_food_to_database so it's there next time.
3. Never web-search a food that appears in the list above "to double check."

SOURCE TAGGING — every food line you show the user (in the confirmation preview) gets a trailing tag:
- 📋 if it matched something already in the food database above.
- 🌐 if you found it via lookup_food_on_web this turn (it has not been added to the database yet, or was just added this turn).
A food already in the database list is always 📋, even if the user has asked about it before. Only a genuinely new lookup this turn gets 🌐.

CONFIRMATION — this is mandatory:
Never call log_food_entries before the user has confirmed. Present your proposed entries as an HTML-formatted message:
- "<b>Log this?</b>" as the header.
- One "• " bullet per food: name, quantity, calories, protein, carbs, fats, then the source tag (📋 or 🌐).
- If you assumed a quantity the user didn't state, add one "<i>(assumed ...)</i>" line after the bullets explaining the assumption.
- A blank line, then "<b>Total:</b> X cal · Yg protein · Zg carbs · Wg fat" for the proposed entries only (not today's running total — you don't have that yet).
- End with: "Reply yes to log, or tell me what to change."
If the user asks for changes, revise and show the same format again. Only log on an explicit yes.

QUANTITIES:
Report quantity as a number plus one of: g, ml, serving. If the user is vague ("a bowl of rice", "some dal"), pick a sensible default, state the assumption plainly (see the <i>assumed...</i> line above), and let them correct it. Do not interrogate them.

ARITHMETIC:
You do not calculate final macros — log_food_entries computes them. For the confirmation preview, scale the per-serving numbers from the list above and present them as estimates.

AFTER LOGGING:
log_food_entries returns today's updated totals (calories, protein, carbs, fats) after the new entries are included. Report, in HTML:
"✅ Logged."
"<b>Today so far:</b> X cal · Yg protein · Zg carbs · Wg fat" — the full running total for the day, as returned by the tool.
"<b>Remaining:</b> X cal · Yg protein" — target minus today-so-far, calories and protein only.

FORMATTING:
Use Telegram HTML tags only: <b>bold</b> for headers and totals, <i>italic</i> only for a stated assumption, "• " for bullet lines, "·" to join values within a totals line. No markdown asterisks or underscores — they will render as literal characters, not formatting.

TONE:
Brief. No preamble, no "Great question!", no emoji beyond the 📋/🌐 source tags and the ✅ on a successful log. This is a utility, not a chat companion.
```

- [ ] **Step 2: Call `validate_node_config` for the agent node with the new `systemMessage`**

Batch call against the single node, confirming the parameter change alone doesn't produce a schema error (e.g. malformed expression braces).

- [ ] **Step 3: Call `update_workflow` to apply only this node's `parameters.options.systemMessage` change**

Prefer a diff-style update over full workflow replacement per CLAUDE.md §2 ("prefer update_workflow diff ops over full replacement").

- [ ] **Step 4: Call `get_workflow_details` and confirm**

Confirm the returned `Calorie Logging Agent` node now has the new `systemMessage` text, and that `connections` is byte-for-byte identical to the Task 1 snapshot (this task should never touch wiring).

---

### Task 4: Stop escaping the agent's own HTML tags in Telegram: Send Message

**Files:**
- Modify (via `update_workflow`): `Telegram: Send Message` node (`id: 66403698-5988-4ec4-89f3-02bffffed0e0`) in workflow `YOUR_WORKFLOW_ID`.

**Interfaces:**
- Consumes: `$json.output` (the agent's final text, now containing intentional `<b>`/`<i>` tags per Task 3).
- Produces: the exact Telegram message body sent to the user; Task 6's manual verification checks this renders as real bold/italic, not literal tags.

- [ ] **Step 1: Replace the `text` expression**

Current:
```
={{ $json.output.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;') }}
```

New — pass the agent's output straight through, since it is the only source of `<`/`>` in this message (no raw user text is interpolated into `output`; user input only reaches the model, never the reply verbatim):
```
={{ $json.output }}
```

- [ ] **Step 2: Call `validate_node_config` for this node with the new `text` value**

- [ ] **Step 3: Call `update_workflow` to apply only this field change**

- [ ] **Step 4: Call `get_workflow_details` and confirm**

Confirm `text` now reads `={{ $json.output }}`, `parse_mode` is still `HTML` under `additionalFields`, and `connections` still matches the Task 1 snapshot.

---

### Task 5: Test with pinned data — database-only log and web-lookup log

**Files:** none (uses `test_workflow`, no file changes)

**Interfaces:**
- Consumes: the updated `Calorie Logging Agent` (Task 3) and `Telegram: Send Message` (Task 4).
- Produces: confirmation this plan's behavior works before relying on a real Telegram round-trip in Task 6.

- [ ] **Step 1: Call `prepare_test_pin_data` (or construct pin data manually) for a text message that matches only foods already in the database**

Pin the `Telegram Trigger` output to a text message like `"200g rice and 1 bowl dal"` where both foods exist in the current food database sheet. Run `test_workflow`.

- [ ] **Step 2: Inspect the `Calorie Logging Agent` output for that run**

Confirm: `<b>Log this?</b>` header present, each food line ends with 📋, a `<b>Total:</b>` line present, no `<i>` line unless a quantity was actually assumed, closing "Reply yes..." line present.

- [ ] **Step 3: Pin a second run for a food NOT in the database**

Use a food description guaranteed absent from the sheet (check the current `foodList` first via the same `get_workflow_details` data, or pick something obscure). Run `test_workflow`.

- [ ] **Step 4: Inspect this run's output**

Confirm the new food's line ends with 🌐, not 📋, and that the agent called `lookup_food_on_web` then `add_new_food_to_database` per the existing LOOKUP ORDER rules (check the execution's tool-call trace, not just the final text).

---

### Task 6: Test the post-log message and verify real HTML rendering in Telegram

**Files:** none

**Interfaces:**
- Consumes: Task 5's confirmed-but-not-yet-logged conversation state; extends it with a confirmation reply.
- Produces: final verification that `log_food_entries`'s returned totals map correctly into "Today so far" / "Remaining", and that Telegram renders `<b>`/`<i>` rather than showing literal tags.

- [ ] **Step 1: Pin a follow-up turn with the user's confirmation ("yes") following Task 5 Step 1's proposal**

Because `Chat Memory` is keyed by Telegram chat ID with a 10-message context window, pinned test runs need to either reuse the same session key or you inject the prior assistant turn into pin data manually — check current pin-data tooling behavior for the memory node before assuming continuity works in `test_workflow` alone.

- [ ] **Step 2: Run `test_workflow` for the confirmation turn**

Confirm the agent calls `log_food_entries` (not before this turn — Task 5's runs must NOT have called it), and that the final message follows the AFTER LOGGING template: `✅ Logged.`, a `<b>Today so far:</b>` line with all four macros, a `<b>Remaining:</b>` line with calories and protein only.

- [ ] **Step 3: Cross-check the numbers**

Compare the "Today so far" values in the message against the raw `log_food_entries` tool-call return value from the same execution — they must match exactly (this task is catching prompt misinterpretation, not sub-workflow math, since the sub-workflow's arithmetic is out of scope for this plan).

- [ ] **Step 4: Send one real Telegram message to the bot (with the user's explicit go-ahead) and confirm rendering**

This is the one step that leaves `test_workflow` pinning and touches the live bot — get explicit confirmation before doing this, per CLAUDE.md §1.4's "ask before execute_workflow" spirit (a live Telegram message triggers the same real-send behavior). Confirm in the actual Telegram client that `<b>`/`<i>` show as bold/italic text, not as literal `<b>` characters, and that 📋/🌐 emoji render normally.

---

### Task 7: Update the workflow log

**Files:**
- Modify: `.claude/WORKFLOW_LOG.md`

**Interfaces:**
- Consumes: outcomes from Tasks 1–6.
- Produces: a new top entry per the file's documented format, so the next session has this change's status without re-deriving it.

- [ ] **Step 1: Prepend a new entry above the existing `## 2026-07-27 — Food Tracker v2` entry**

Follow the file's own format block exactly:

```
## <today's date> — Food Tracker v2 — Reply Formatting
- Workflow ID: `YOUR_WORKFLOW_ID`
- Change: Rewrote the Calorie Logging Agent system prompt to emit HTML-formatted
  replies (bold headers/totals, bullet lines, italic assumption notes) and to
  tag every food line with its source (📋 database match, 🌐 fresh web lookup).
  Post-log messages now report both "Today so far" (all four macros) and
  "Remaining" (calories/protein), not remaining alone. Removed the HTML-escaping
  on Telegram: Send Message's output expression, which had been stripping the
  agent's own <b>/<i> tags.
- Why: Replies were plain and gave no indication of whether a food's macros
  came from the trusted local database or an unverified web search.
- Status: <fill in based on actual Task 5/6 outcomes: tested / published>
- Next: <fill in anything Task 5/6 surfaced that still needs following up>
```

- [ ] **Step 2: Save the file**

---

## Self-Review Notes

- Spec coverage: source tagging (Task 3), confirmation template (Task 3), post-log template (Task 3), escaping fix (Task 4), testing both source types (Task 5) and the post-log path (Task 6), rollback snapshot (Task 1) — all spec sections have a task.
- The spec's rejected "pre-log running total" is correctly absent from the confirmation template in Task 3.
- No task references a function/parameter name not confirmed in Task 2 or drawn directly from the existing `get_workflow_details` snapshot already on hand.
