# CLAUDE.md — n8n Workflow Engineering

Rules for building, editing, and debugging n8n workflows on my self-hosted instance via the
connected n8n MCP server. These are enforceable defaults, not suggestions. Violating one
requires saying so out loud and why.

---

## -1. Session continuity — read this first

Read `.claude/WORKFLOW_LOG.md` at the start of any session before asking what we were
working on — it's the running record of workflows built/changed here. After any
create/update/publish action (or a meaningful debugging session), append a new entry
following the format at the top of that file. Don't wait to be asked.

**Local version control (community plan has no built-in workflow history):** before
calling `update_workflow` on any existing workflow, first call `get_workflow_details` on
it and save the full node/connections JSON to
`workflow_versions/<Workflow_Name_With_Underscores>/<YYYY-MM-DD_HHmm>_<short-label>.json`
(project root, not inside `.claude/`)
— label describing the state being left behind, e.g. `pre-add-retry-logic`. Never
overwrite an existing snapshot file. This is the only rollback path that exists, so treat
it as mandatory, not optional — a missed snapshot means a change can't be undone. After
`create_workflow_from_code` for a brand-new workflow, save the initial result the same
way with label `baseline`.

To roll back: read the desired snapshot file, then `update_workflow` with that
`nodes`/`connections` content, then verify via `get_workflow_details` before publishing.

---

## 0. Tool surface — read this before anything

My server exposes the **official n8n Workflow SDK** surface. The `n8n-*` skills installed in
this project describe the **community `n8n-mcp`** surface. Their *domain rules are correct*;
their *tool names are not*. Never call a tool a skill names without checking it exists.

| Need | Live tool |
|---|---|
| Plan | `get_workflow_best_practices` (`technique="list"` if unsure) |
| SDK syntax | `get_sdk_reference` — **required before writing code** |
| Find nodes | `search_nodes` |
| Exact params | `get_node_types` — **required before configuring** |
| Check one node | `validate_node_config` (free, read-only, batch up to 50) |
| Check the graph | `validate_workflow(code)` |
| Ship | `create_workflow_from_code` / `update_workflow` |
| Inspect | `get_workflow_details`, `search_workflows`, `get_execution`, `search_executions` |
| Run | `test_workflow` (pin data) → `execute_workflow` (real side effects) |
| Activate | `publish_workflow` |
| Resource lookups | `explore_node_resources`, `list_credentials`, `search_projects`, `search_folders` |

There is no `n8n_create_workflow`, `n8n_update_partial_workflow`, `get_node`, or
`n8n_test_workflow` here. If a skill names one, translate to the table above and tell me
about the drift.

---

## 1. Non-negotiables

1. **Never guess syntax or parameter names.** `get_sdk_reference` before code,
   `get_node_types` before parameters. Remembered names validate as plain strings and then
   do nothing at runtime — the worst failure mode.
2. **Never activate on validation alone.** `validate_workflow` proves the JSON is well-formed,
   not that the logic is right. Always `get_workflow_details` after create/update and read the
   `connections` object with your own eyes before `publish_workflow`.
3. **Secrets only via `newCredential('Name')`.** No API keys in Set nodes, no invented
   credential IDs, no `mock-*` values. A token in `{{ $json.token }}` is a leak with extra steps.
4. **Ask before `execute_workflow`.** It fires real sends, writes, and charges. `test_workflow`
   with pin data first.

---

## 2. Build loop

```
Requirements → get_workflow_best_practices → search_nodes → get_node_types
→ validate_node_config (per node, as you write) → write SDK code
→ validate_workflow → create/update → get_workflow_details (verify connections)
→ test_workflow → publish_workflow
```

- Ask me clarifying questions before building anything non-trivial. Trigger, data shape,
  volume, failure behaviour, and where output lands are the five I always forget to state.
- Search my existing workflows before building new logic — `search_workflows`.
- Prefer `update_workflow` diff ops over full replacement. Never rebuild a live workflow from
  scratch to fix one field.
- Default timezone **Asia/Kolkata**. Set it explicitly on every schedule and date expression.
  n8n shows green while running on the wrong clock.

---

## 3. Structural rules (SDK)

- **Indices are 0-based.** `.input(0)` is the FIRST input. Wiring N branches into Merge uses
  `0..N-1`. Starting at `1` silently drops branch one. Merge also defaults to 2 inputs — a
  third source vanishes unless you raise the count.
- **Empty item lists are correct.** Zero items means "nothing to do this round." Never add
  `alwaysOutputData: true` to keep a chain alive, and never gate a loop with an IF that checks
  "has items?" — `splitInBatches`, `filter`, and per-item nodes already no-op on empty input.
  `alwaysOutputData` is only for a deliberate "no matches found" branch, paired with an IF.
- **`executeOnce: true` on any node that should fire once, not per item.** Summary
  notifications, report generation, shared-context fetches. Duplicate Slack messages are
  almost always a missing `executeOnce`.
- **Control-flow primitive selection:**
  per-item work with side effects → `splitInBatches(batchSize: 1)` + `nextBatch`;
  drop non-matching items → `filter`; two real paths → `IF`; many keyed paths → `switch`.
  A Filter/IF selects — it does not perform the action. Wire the action node on the branch.
- **Per-item iteration is automatic.** Don't add Loop Over Items to "make it loop."

---

## 4. Expressions

- **Webhook data is under `.body`.** `{{ $json.body.email }}`, never `{{ $json.email }}`.
  This is the single most common broken-workflow cause.
- **In branchy flows prefer `$('Node Name').item.json.x` over `$json.x`.** `$json` breaks
  silently the moment a node is inserted, or after Aggregate / Code-run-all / merges.
- **Delete Set nodes feeding 0–1 consumers.** Inline the expression at the consumer. Keep a
  Set only for 2+ consumers of a non-trivial derived value, output-shaping at a sub-workflow
  boundary, or deliberate field whitelisting.
- **Anchor branch convergence with a named NoOp** (`Combine Inputs`) and reference it by name
  downstream. Use a Set instead when branches produce different shapes.
- **Luxon inline beats a DateTime node** for date math. `{{ $now.minus({days: 7}).toISO() }}`.
- No expressions in webhook paths or credential fields.

---

## 5. Error handling

- **Per-node error routing is TWO steps.** Set `onError: "continueErrorOutput"` *and* wire
  `main[1]` to a handler. `onError` alone silently discards errors and the dashboard reports
  **success**. A wired output alone never fires.
- **`retryOnFail` before you wire error paths.** Transient 429/timeout should self-heal, not
  page anyone.
- **Every webhook, scheduled, and unattended workflow gets an error path.** Happy-path-only is
  not done.
- **Status codes map to cause, not convenience.** 4xx is decided *before* the work by an
  upstream IF/validator (400 validation, 401/403 auth, 404 lookup miss, 409 conflict, 429
  rate limit). 5xx comes *out of* error outputs (500 unknown, 502 upstream, 503 unavailable,
  504 upstream timeout). One expression-driven Respond node unless bodies differ structurally.
- Envelope: `{ "error": "<code>", "message": "<human text>" }`. Never leak stack traces, SQL,
  upstream bodies, or tokens — log those privately.

---

## 6. AI agents

- **Tool names and descriptions ARE the prompt.** The model selects on those alone. `tool1`
  with an empty description is invisible; there is no error, just an agent that "won't use my
  tool." Write them like API docs.
- **Structured output needs `autoFix: true` + a coding-capable fixer model.** Without it one
  malformed JSON halts the workflow.
- **Raise `options.maxIterations`.** The default is single digits. 15 for a focused sub-agent,
  50–200 for an orchestrator. "Max iterations reached" or empty output = this.
- **`$fromAI('name', 'specific description', 'type')`** for parameters the model should decide.
  Plumb identity, limits, and correlation IDs (`userId`, refund caps, `sessionId`)
  deterministically — the agent shouldn't see them. `$fromAI` cannot carry binary.
- **Default to `.toolWorkflow`** for anything multi-step or reusable. Native tool node for
  single-operation jobs. `.toolCode` has a different runtime contract — read `n8n-code-tool`.
- **Gate destructive tools behind human review.** Sends, payments, refunds, account changes.
- Put `{{ $now }}` in the system prompt. Hardcoded dates are stale on day one.

---

## 7. Code nodes

Last resort, in this order: expression → arrow function in Edit Fields → Code node.
JavaScript by default; Python only if I explicitly ask. If a Code node exists, it should have
a one-line comment saying why an expression couldn't do it.

---

## 8. Domain profiles

- **Ecommerce / marketing:** dedupe before acting (order IDs replay). Idempotency keys on
  every send. Cap batch sizes — a 5k-row loop that fires per item will rate-limit and
  double-charge. Suppression lists are a `filter`, not an IF.
- **Scheduled reports & pipelines:** explicit timezone, explicit lookback window
  (`$now.minus(...)`), `executeOnce` on the summary/send node, and empty-result handling that
  reports "no data" rather than crashing on `undefined`.
- **Webhook & API:** validate input upstream and return 400 there. Respond fast, do heavy work
  after the Respond node or in a sub-workflow. Auth on every webhook — `n8n_audit_instance`
  equivalents flag unauthenticated ones.
- **Chatbots:** `sessionId` from a stable user key, never the message ID. Images need a CDN
  URL, not base64 through the agent boundary.

---

## 9. Antipatterns — stop if you catch yourself doing these

| Doing | Instead |
|---|---|
| "It's simple, I'll just build it" | Most "simple" flows ship at 10+ nodes. Plan first. |
| Set node to map one field | Inline at the consumer |
| Code node because it's easier | Expression → Edit Fields → then Code |
| `alwaysOutputData` to keep a chain alive | Trust the empty list |
| Merge inputs starting at `1` | 0-based: `0..N-1` |
| `onError` set, error output unwired | Wire `main[1]` or the failure is invisible |
| Publishing because validation passed | Read `connections`, then test, then publish |
| Copy-pasting logic into a second workflow | Extract a sub-workflow |

---

## 10. Definition of done

Validated → connections verified by reading them → tested with pin data → error paths wired →
credentials via `newCredential` → timezone explicit → and you've told me which project/folder
it landed in and what will happen the first time it fails.
