# Email Classification Workflow — AI-Powered Automated Routing

An n8n automation that reads incoming Gmail messages, classifies them into business departments using AI, and routes them in real time to Slack and Airtable — fully hands-off once active.

Built as part of the TS Academy AI Automation Phoenix Cohort, then extended beyond the base assignment to demonstrate production-style patterns: structured AI output, multi-channel notification, and persistent logging.

## What it does

1. Watches a Gmail inbox for new unread emails
2. Pulls the full message and cleans it into a consistent shape
3. Sends it to an AI Agent (Groq, Llama 3.3 70B) that classifies the email into one of six categories and extracts sender, subject, and a one-sentence summary — returned as validated JSON via a Structured Output Parser
4. Routes the classified email to one of six branches based on category: **Sales, Customer Service, HR, Finance, Operations,** or **Fallback**
5. On each branch: posts a formatted notification to Slack, logs the full record to Airtable, and marks the original email as read so it isn't reprocessed

## Tech stack

- **n8n** (self-hosted on Railway) — workflow orchestration
- **Gmail** — trigger + message retrieval
- **Groq** (Llama 3.3 70B Versatile) — AI classification
- **Slack** — real-time department notifications
- **Airtable** — persistent record of every classified email

## Architecture

```
Gmail Trigger → Get Message → Code (clean data) → AI Agent + Structured Output Parser
                                                              ↓
                                                        Switch (6-way route)
                                                              ↓
                            ┌──────────┬──────────┬─────┬─────────┬────────────┐
                          Sales   Customer Svc     HR   Finance  Operations  Fallback
                            ↓          ↓           ↓       ↓         ↓          ↓
                          Slack → Airtable → Mark as Read   (same pattern on every branch)
```

## Key technical decisions & problems solved

- **Structured output over free-text parsing.** Early versions had the AI return a single classification word, which was fragile — any deviation broke the Switch node. Adding a Structured Output Parser forces valid JSON with a defined schema (`category`, `from`, `subject`, `summary`), making downstream routing and logging reliable.
- **Node-scoped data references.** Because the AI Agent replaces the item's JSON with its own output, fields like the original send date or sender don't survive downstream by default. Solved by explicitly referencing upstream nodes (e.g. `$('Code in JavaScript').item.json.date`) rather than assuming `$json` still holds them.
- **Airtable typecasting.** Airtable's Date field rejected ISO date-time strings from Gmail's epoch timestamp. Fixed by converting to a clean date-only string and enabling Airtable's Typecast option for forgiving type coercion.
- **Markdown escaping in Slack/Telegram.** MarkdownV2 requires escaping reserved characters (like `-` in email addresses), which broke on real-world sender addresses. Resolved by using standard Markdown formatting instead.
- **Read-state hygiene.** Added a Mark as Read step on every branch so the poll-based trigger never reprocesses the same email twice.

## Known limitation / future improvement

The current design assigns exactly one category per email. A real inbox sometimes contains messages that genuinely span multiple departments (e.g. a refund request bundled with a bulk pricing question). Right now, the whole email routes to whichever category the AI judges dominant — other relevant departments never see it.

**Planned v2 approach:** extend the Structured Output Parser schema to return an array of department-specific segments, each containing only the department-relevant excerpt, followed by a Split Out node so each department receives an isolated item with no visibility into what other departments received. This is a data-separation approach rather than a permissions system — it depends on the AI's ability to correctly identify topic boundaries, which is a reasonable default but not a hard guarantee for highly sensitive content.

## Screenshots

*(See linked Google Drive folder for full screenshot set: workflow canvas, node configs, execution logs, Slack notifications, Airtable records.)*

## Author

Doreen Akpede — Business Operations & AI Automation
[LinkedIn](https://linkedin.com/in/onohinosen-akpede) · [GitHub](https://github.com/doreen-akpede)
