# Food-Tracker

A Telegram bot that logs what you eat into Google Sheets and tracks your daily macros. Built as n8n workflows.

## The Problem

People who track calories and macros hit the same walls:

- Calorie apps need many taps per meal: search, pick a match, set a portion, save.
- Food databases in those apps are often wrong or crowd-edited, and you cannot tell which entries to trust.
- Photo logging in most apps is a paid feature or a guess with no way to check it.
- Your data stays inside the app. A spreadsheet you control is hard to keep in sync by hand.
- Chat-based AI helpers forget what you ate earlier and do arithmetic in the model, which gets it wrong.

## The Solution

You message a Telegram bot with text or a photo of your meal. Then:

1. The bot receives the message. If it has a photo, a vision model describes the food as plain text.
2. The workflow loads your food database and your weekly macro goals from Google Sheets.
3. An AI agent matches the foods you ate to rows in your database.
4. If a food is not in the database, the agent looks it up on the web and adds it as a new row.
5. A separate workflow computes the macros with code and writes the entries to the tracking sheet.
6. The bot replies with what it logged, your totals so far today, and what is left.

## How It Helps

- Logging a meal is one message, or one photo, instead of a search-and-confirm loop.
- Every food line in the reply is tagged as a database match or a fresh web lookup, so you know which numbers are verified.
- Macro math runs in a Code node, not in the AI model. The agent only decides what to log.
- Your data lives in your own Google Sheet. You can edit or delete rows by hand.
- Daily totals are re-summed from the raw rows on every log, so a row you delete is reflected the same day.
- Failures send you a Telegram message instead of failing silently.

## What Makes It Different

| Feature | How it works here |
|---|---|
| Single agent | One AI agent with three tools, instead of a chain of separate LLM calls |
| Database in the prompt | The full food list is loaded into the prompt, so there is no lookup tool for known foods |
| Source tags | Each logged food is marked as database or web lookup |
| Strict food IDs | The logging sub-workflow throws if a food ID is not in the database, so the agent must add unknown foods first |
| Photo input | A separate vision model turns a photo into a text description |
| Memory | The last 10 messages are kept per Telegram chat |
| Error alerts | A catch-all workflow messages you with the failed node and error text |

## How It Works

    Telegram message
           |
           v
     [Has Photo?]--yes--> Get File --> Vision model (describe food)
           |                                   |
           no                                  |
           v                                   v
     Format Text Input                  Format Vision Input
           \                                   /
            \_____________  ___________________/
                          \/
                    Combine Inputs
                          |
        Read Food Database + Read Weekly Goals (Google Sheets)
                          |
                  Build Prompt Data (Code)
                          |
                          v
                 Calorie Logging Agent (Gemini)
                 |            |             |
          lookup_food_   add_new_food_   log_food_entries
          on_web         to_database     (sub-workflow)
          (Tavily)       (Sheets)             |
                                    Read food DB -> Build rows
                                    -> Append tracking row
                                    -> Re-sum today -> Upsert totals
                          |
                          v
                 Telegram: Send Message

    Sheet read fails --> Notify Data Fetch Failure (Telegram)
    Any other failure --> Error Alerts workflow (Telegram)

## Tech Stack

| Part | Used for |
|---|---|
| n8n (self-hosted) | Runs the workflows |
| Telegram Bot API | Chat interface for text and photos |
| Google Gemini 2.5 Flash | Agent model and photo description |
| Google Sheets | Food database, weekly goals, tracking rows, daily totals |
| Tavily | Web lookup for foods not in the database |
| JavaScript (n8n Code nodes) | Prompt data, macro math, summaries |

## Run It Locally

This repo holds workflow snapshots and notes, not an app. There is no package to install.

Needs: a running n8n instance, a Telegram bot token from BotFather, a Google account, a Gemini API key, a Tavily API key.

1. Create a Google Sheet with four tabs: `food database`, `weekly macro goals`, `tracking table`, `total daily macros tracking table`.
2. In n8n, create these credentials: Telegram API, Google Sheets OAuth2, Google Gemini (PaLM) API, Tavily API.
3. Import the JSON files from `workflow_versions/` as new workflows:
   - `Food_Tracker_v2/2026-07-27_baseline.json` (main workflow)
   - `Log_Food_Entries_sub_workflow/2026-07-27_baseline.json` (sub-workflow)
   - `Food_Tracker_v2_Error_Alerts/2026-07-27_baseline.json` (error alerts)
4. Replace the placeholders in the nodes:
   - `YOUR_GOOGLE_SHEET_ID` with your sheet ID
   - `YOUR_TELEGRAM_CHAT_ID` with the chat that should get error alerts
   - `YOUR_WORKFLOW_ID` with the ID of your imported sub-workflow
   - Re-pick the tab in each Google Sheets node, since tab IDs differ per sheet.
5. Attach your credentials to each node that needs one.
6. In the main workflow settings, set "Error Workflow" to the error alerts workflow.
7. Activate the workflows and send your bot a message.

The snapshots are stored node and connection data. They have not been tested as a fresh import. Expect to fix some node fields by hand.

There are no environment variables. All keys are stored as n8n credentials. Do not paste keys into nodes.

### Warnings

- The Telegram trigger has no user allow-list. Anyone who finds your bot can send it messages, use your Gemini and Tavily quota, and write to your sheet. Add a check on the chat ID before the agent runs.
- Gemini and Tavily are paid or quota-limited APIs. Photo messages cost more than text.
- Web lookups are unverified. Check the numbers on any food tagged as a web lookup.
- The Google Sheets credential gives n8n access to your sheet. Use a sheet that holds only this data.
- Never commit real credentials, bot tokens or chat IDs. The files in this repo use placeholders.

## Status

Done:
- Main agent workflow, logging sub-workflow, and error alerts workflow built and published.
- Reply formatting (HTML, source tags, totals and remaining) tested and confirmed in Telegram.
- Daily totals re-summed from raw rows on each log. Tested with pinned data.

Next:
- Confirm the main workflow's Error Workflow setting points to the error alerts workflow, then force a failure to check the alert.
- Test the "unknown food ID" error end to end through the agent.
- Add a chat ID allow-list to the Telegram trigger.
- Handle deleting the only log of a day, which the logging flow does not recompute.

Notes and change history are in `.claude/WORKFLOW_LOG.md`. Design docs are in `docs/superpowers/`.
