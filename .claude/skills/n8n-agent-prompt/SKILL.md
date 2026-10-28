---
name: n8n-agent-prompt
description: Build production-grade system prompts for n8n AI Agent nodes. Use this skill whenever the user wants to create, improve, debug, or rewrite a system prompt for an n8n AI agent workflow — including WhatsApp bots, lead qualification agents, email agents, support agents, data processing agents, or any automation that uses an AI Agent node in n8n. Also trigger when the user says things like "write me an agent prompt", "build a system prompt for my workflow", "my n8n agent isn't responding correctly", "help me prompt engineer my automation", or mentions AI agents in the context of n8n, Make, or workflow automation. If the user pastes an existing n8n system prompt and asks for feedback, improvements, or debugging — use this skill.
---

# n8n Agent Prompt Builder

Build system prompts for n8n AI Agent nodes that are deterministic, grounded, and production-ready.

## Why most n8n agent prompts fail

n8n AI Agent prompts are not the same as chatbot prompts. They fail for specific, recurring reasons:

1. **They describe the agent's personality instead of its behavior.** "You are a helpful assistant" does nothing. The agent needs to know what to do when it receives a message, what tools it has, and what output format downstream nodes expect.

2. **They assume tools that don't exist.** The prompt says "search LinkedIn" but no LinkedIn tool is connected. The agent hallucinates actions it cannot perform.

3. **They have no error handling.** When a tool returns empty results or an API times out, the agent either crashes silently or makes up data.

4. **They don't account for n8n's execution model.** n8n agents run in a single execution context. They don't persist memory between executions unless you explicitly wire it. Prompts that assume conversation history without a memory tool will produce inconsistent behavior.

5. **They let the AI decide the output format.** Downstream nodes (Google Sheets, HTTP Request, webhooks) expect structured data. If the agent returns prose instead of JSON, the workflow breaks.

## How this skill works

### Step 1: Assess the user's input

Determine which mode to operate in:

**Structured mode** — The user provides enough detail to generate immediately. They've told you:
- What the agent does (objective)
- What triggers it (webhook, schedule, manual)
- What tools are connected (or what APIs it calls)
- What happens after the agent responds (downstream nodes)

→ Skip the interview. Generate the prompt directly.

**Interview mode** — The user gives a vague request like "build me a lead qualification agent" or "I need a prompt for my WhatsApp bot." Key details are missing.

→ Ask focused questions. Do NOT dump a generic questionnaire. Ask only what's missing from this list:

1. **Trigger**: What kicks off this agent? (Webhook from WhatsApp? Incoming email? Scheduled cron? Manual?)
2. **Objective**: What is the agent supposed to accomplish in a single execution?
3. **Tools available**: What tools/sub-nodes are connected to the AI Agent node? (HTTP Request, Google Sheets, code nodes, etc.)
4. **Input shape**: What does the incoming data look like? (WhatsApp message body? Email subject + body? A JSON payload?)
5. **Output destination**: What node comes after the agent? What format does it need? (JSON for a Google Sheets append? Plain text for a reply message? A webhook response?)
6. **Failure behavior**: What should happen if a tool fails or returns no data?
7. **Scope boundaries**: What should the agent refuse to do or hand off to a human?

Ask a maximum of 3 questions per message. Prioritize the gaps that would cause the biggest problems if left undefined.

### Step 2: Generate the system prompt

Use this structure. Every section exists for a reason — do not skip sections, but adapt the content to the specific use case.

```
IDENTITY AND SCOPE
[Who the agent is. One sentence. Grounded in the specific workflow, not generic.]

OBJECTIVE
[What the agent must accomplish per execution. Be specific and measurable.]

EXECUTION CONTEXT
- Trigger: [What starts this execution]
- Input format: [Exact shape of incoming data]
- Output destination: [What node receives the agent's response]
- Required output format: [Exact structure — usually JSON schema or specific text format]

AVAILABLE TOOLS
[List every tool connected to the AI Agent node. For each tool, specify:]
- Tool name: [as it appears in n8n]
- What it does: [one line]
- When to use it: [specific trigger condition]
- Expected input: [what the agent should pass to it]
- Expected output: [what comes back]

DECISION LOGIC
[Step-by-step instructions for how the agent should process a request. This is the core of the prompt. Write it as a numbered sequence with explicit branching.]

1. Parse the incoming [message/payload/email].
2. Extract [specific fields].
3. If [condition], use [Tool A] to [action].
4. If [Tool A] returns empty/error, [fallback behavior].
5. Based on the result, determine [classification/response/action].
6. Format the output as [exact format specification].

OUTPUT FORMAT
[Provide the exact JSON schema, template, or structure the agent must return. Include an example.]

Example output:
{
  "field_1": "value",
  "field_2": "value",
  "action": "value"
}

RULES
- [Constraint 1: what the agent must never do]
- [Constraint 2: scope limitation]
- [Constraint 3: data handling rule]
- Never fabricate data. If a tool returns no results, say so explicitly.
- Never respond outside the defined output format.
- If the request is outside your scope, respond with: {"status": "escalate", "reason": "[brief reason]"}

ERROR HANDLING
- If [Tool X] fails or times out: [specific fallback]
- If required input fields are missing: [specific response]
- If the request is ambiguous: [ask for clarification OR make a safe default choice — specify which]
```

### Step 3: Generate test cases

After generating the system prompt, always provide 3 test cases:

**Test Case 1: Happy path**
- Input: A realistic, well-formed input that the agent should handle perfectly.
- Expected output: The exact output the agent should produce.

**Test Case 2: Edge case**
- Input: A tricky but realistic input (missing fields, unusual formatting, ambiguous request).
- Expected output: How the agent should handle it gracefully.

**Test Case 3: Out-of-scope request**
- Input: Something the agent should refuse or escalate.
- Expected output: The refusal/escalation response in the correct format.

Format each test case as:
```
TEST CASE [N]: [Descriptive name]
---
Input:
[The exact input the agent would receive]

Expected Output:
[The exact output the agent should produce]

Why this tests well:
[One sentence explaining what this validates]
```

### Step 4: Provide n8n implementation notes

After the prompt and test cases, include a brief "Implementation Notes" section:

- Which n8n AI Agent type to use (Tools Agent vs. ReAct Agent vs. Conversational Agent) and why
- Whether memory should be connected (and what type: window buffer, summary, etc.)
- Any specific tool configuration notes (e.g., "Set the HTTP Request node to return the full response including headers")
- Output parsing tips if the downstream node needs specific formatting

## Quality checklist

Before delivering the final prompt, verify:

- [ ] Every tool mentioned in the prompt is actually available in n8n (or is a connected sub-node)
- [ ] The output format matches what the downstream node expects
- [ ] There is explicit error/fallback handling for every tool
- [ ] The decision logic has no ambiguous branches (every "if" has an "else")
- [ ] The prompt does not assume persistent memory unless a memory node is connected
- [ ] Scope boundaries are defined (what the agent will NOT do)
- [ ] At least one test case covers a failure/edge scenario
- [ ] The prompt uses concrete field names, not vague references like "the relevant data"

## Common n8n agent patterns

Reference these when building prompts for common use cases. Read `references/patterns.md` for detailed templates.

- **Webhook responder**: Agent receives a webhook, processes it, returns a structured response
- **Lead qualifier**: Agent receives a message, asks qualifying questions, scores the lead
- **Email processor**: Agent reads incoming email, classifies it, drafts a response or routes it
- **Data enrichment**: Agent receives a record, uses tools to enrich it, returns the enriched record
- **Conversational bot**: Agent maintains context via memory node, handles multi-turn dialogue
- **Content generator**: Agent receives a brief, generates content, returns it in a specific format

## Anti-patterns to avoid

When reviewing or generating prompts, flag these:

- **"Be creative"** — Creativity is the opposite of what you want in an automation agent. Be deterministic.
- **"Use your best judgment"** — The agent will use different judgment every time. Define the criteria explicitly.
- **"If you're unsure, do your best"** — This guarantees inconsistent behavior. Define the fallback.
- **"You have access to the internet"** — No, it has access to specific tools. Name them.
- **Personality paragraphs** — Three sentences about being friendly and professional waste tokens and change nothing.
- **Unbounded output** — If you don't constrain the output format, every downstream node is gambling.
