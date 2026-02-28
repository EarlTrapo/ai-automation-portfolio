# 🤖 Telegram Broker Registration Bot — AI + Chatwoot + Human Takeover

A production-grade n8n automation that handles broker account registration inquiries via Telegram, powered by an Anthropic Claude AI agent, fully integrated with Chatwoot CRM, and built with a complete human escalation system.

---

## 📌 What This Solves

Financial services businesses that handle broker onboarding face a repetitive, high-volume problem: users ask the same classification and registration questions over and over, but some edge cases genuinely need a human. This workflow automates the 80% that's routine while ensuring the 20% that's complex reaches a real agent — without losing any conversation context in the handoff.

---

## 🏗️ Architecture Overview

The system has two distinct execution paths that run in parallel:

**Path 1 — Inbound User Messages (Telegram → AI → Chatwoot)**
Handles every new message from a user on Telegram, routes it through the AI agent or directly to a human depending on the takeover flag, and logs everything.

**Path 2 — Human Agent Replies (Chatwoot → Telegram)**
A separate webhook listener catches outgoing messages from human agents in Chatwoot and forwards them back to the user's Telegram chat in real time.

---

## 🔁 Full Flow Walkthrough

### Trigger & Classification
1. **Telegram Trigger** — listens for incoming messages from users
2. **Extract Message Fields** — parses `chat_id`, `text`, `first_name`, `last_name`, `username` from the raw Telegram payload
3. **Check Human Takeover Flag** — reads a Google Sheet row for this user to check if a human agent has taken over the conversation
4. **IF: Human Takeover Active?** — branches the flow

---

### Branch A — Human Takeover is ON
If an agent has flagged this conversation for manual handling:

5. **Mirror User Msg to Chatwoot (Takeover)** — forwards the user's message directly into the Chatwoot conversation thread so the human agent sees it in real time. The AI does **not** respond.

---

### Branch B — AI Handles the Conversation

5. **Merge Context Data** — combines message fields with the user's stored data from Sheets (conversation ID, contact ID, current status)
6. **Search Chatwoot Contact** — queries Chatwoot API to look up the user by Telegram chat ID
7. **IF: Contact Exists?** — checks if this is a returning user

   - **New user** → **Create Chatwoot Contact** with name and Telegram metadata  
   - **Returning user** → passes through directly

8. **Resolve Contact ID** — extracts the Chatwoot contact ID regardless of which branch was taken
9. **IF: Conversation Exists?** — checks if an open conversation thread already exists for this user

   - **No conversation** → **Create Chatwoot Conversation** — opens a new thread and stores `telegram_chat_id` in the conversation's `additional_attributes` for later reverse-lookup
   - **Existing** → passes through

10. **Resolve Conversation ID** — extracts the final conversation ID
11. **Mirror Incoming Msg to Chatwoot** — logs the user's message into the Chatwoot conversation for full audit trail
12. **Log User Message to Sheets** — appends a row to Google Sheets with timestamp, user info, and message content

---

### AI Agent Processing

13. **AI Agent** (n8n LangChain agent) — powered by **Anthropic Claude**, guided by a detailed system prompt that:
    - Classifies users as **EU/EEA residents** or **Non-EU residents** based on tax residency, physical residency, and citizenship
    - Walks EU users through the **MiFID II compliant** registration path
    - Walks Non-EU users through the **alternative broker** registration path
    - Emits a `[STATUS: EU | NON_EU | REGISTERED | IN_PROGRESS]` tag in every response
    - Emits a `HUMAN_NEEDED: <reason>` signal when escalation is required

14. **Window Buffer Memory** — maintains short-term conversation memory scoped to the session so the AI has context across multiple messages

15. **Parse Agent Output** — custom code node that:
    - Strips `[STATUS:...]` tags and `HUMAN_NEEDED:...` lines from the user-facing reply
    - Extracts the status and escalation reason into structured fields
    - Returns a clean `cleanReply` for Telegram delivery

---

### Routing: Normal Reply vs. Escalation

16. **Route: Escalation or Normal** — switch node that checks `needsHuman`

**Normal path:**
17. **Update Chatwoot Labels** — applies the status tag (e.g. `IN_PROGRESS`, `REGISTERED`) as a label on the Chatwoot conversation
18. **Send Telegram Reply** — delivers the cleaned AI response to the user
19. **Mirror AI Reply to Chatwoot** — logs the AI's response in Chatwoot for the full conversation record
20. **Log AI Reply to Sheets** — appends the AI reply to the Google Sheets audit log

**Escalation path:**
17. **Flag Chatwoot for Human** — posts a note to Chatwoot and updates the conversation status to indicate human review needed
18. **Set Takeover Flag in Sheets** — flips `takeover_active` to `true` for this user's row so future messages bypass the AI
19. **Telegram: Escalation Notice** — sends the user a message letting them know they're being connected to a human agent

---

### Human Agent Reply Path (Chatwoot Webhook)

This runs as a completely separate trigger:

20. **Chatwoot Webhook (Human Reply)** — listens for `message_created` events from Chatwoot
21. **Filter & Extract Human Reply** — validates that the event is:
    - An outgoing message (sent by an agent, not the user)
    - Not an AI-generated message
    - Contains a valid `telegram_chat_id` in the conversation attributes
22. **Forward Human Reply via Telegram** — sends the agent's message to the user's Telegram chat
23. **Log Human Reply to Sheets** — appends the agent reply to the Google Sheets log
24. **Chatwoot Webhook Response** — returns a 200 OK to Chatwoot to acknowledge receipt

---

## 🧰 Tech Stack

| Layer | Tool |
|---|---|
| Automation runtime | n8n |
| AI model | Anthropic Claude (via n8n LangChain integration) |
| Messaging | Telegram Bot API |
| CRM / Live chat | Chatwoot |
| Memory | n8n Window Buffer Memory |
| Logging & state | Google Sheets |

---

## 📂 Repository Structure

```
ai-automation-portfolio/
├── workflows/
│   └── Telegram_Broker_Registration_Bot_with_AI_Chatwoot_Human_Takeover.json
├── screenshots/
│   └── workflow-canvas.png
└── README.md
```

---

## ⚙️ Setup & Requirements

### Prerequisites
- n8n instance (self-hosted or cloud)
- Telegram Bot (via [@BotFather](https://t.me/botfather))
- Chatwoot account with API access
- Anthropic API key
- Google Sheets with the following columns:
  - `chat_id` | `takeover_active` | `chatwoot_conversation_id` | `chatwoot_contact_id` | `current_status`

### Required n8n Credentials
- **Telegram** — Bot API token
- **Anthropic** — API key (connected to the LangChain Anthropic Chat Model node)
- **Chatwoot** — API token + base URL
- **Google Sheets** — OAuth2 or Service Account

### Chatwoot Webhook Configuration
In Chatwoot → Settings → Integrations → Webhooks, add your n8n webhook URL and enable the `message_created` event.

### Google Sheets State Management
The workflow uses one Sheet as a simple state store per user. Each row represents one Telegram user and tracks:
- Whether a human has taken over (`takeover_active: true/false`)
- The Chatwoot contact and conversation IDs for fast lookups without repeated API searches
- Current registration status

---

## 🔐 Key Design Decisions

**Why mirror messages to Chatwoot even when the AI is handling it?**  
So agents always have a full, real-time view of every conversation. When escalation happens, there's zero context gap — the agent can scroll up and see the entire history.

**Why use Google Sheets for state instead of a database?**  
Sheets gives non-technical operators visibility and direct edit access to override flags (e.g. manually flipping `takeover_active` back to `false` after an agent resolves an issue) without needing any backend access.

**Why does the AI embed `[STATUS:]` tags in its own output?**  
It removes the need for a separate classification call. The AI does the reasoning and signals its conclusion inline, which the Parse Agent Output node extracts programmatically. One LLM call does both jobs.

**Why two separate triggers?**  
The inbound (Telegram) and outbound (Chatwoot webhook) paths have completely different lifecycles and don't need to share state mid-execution. Keeping them as separate flows prevents timeout issues and makes debugging significantly easier.

---

## 📊 Data Flow Diagram

```
User (Telegram)
      │
      ▼
[Telegram Trigger]
      │
      ├─ Check takeover flag ──► Takeover ON ──► Mirror to Chatwoot only
      │
      └─ Takeover OFF
            │
            ▼
      Search/Create Chatwoot Contact & Conversation
            │
            ▼
      Mirror message to Chatwoot + Log to Sheets
            │
            ▼
      AI Agent (Claude) + Window Buffer Memory
            │
            ▼
      Parse output → Route decision
            │
            ├─ Normal ──► Update labels ──► Reply on Telegram ──► Mirror to Chatwoot ──► Log
            │
            └─ Escalation ──► Flag Chatwoot ──► Set Sheets flag ──► Notify user on Telegram


Chatwoot (Agent sends reply)
      │
      ▼
[Chatwoot Webhook Trigger]
      │
      ▼
Filter: outgoing + not AI + has telegram_chat_id
      │
      ▼
Forward to user on Telegram + Log to Sheets + Respond 200
```

---

## 👤 Author

Built by [Elvis] — AI Systems Optimizer specializing in automation workflows, custom AI integrations, and production-grade chatbot systems.

> Part of the `ai-automation-portfolio` — a collection of real-world n8n workflows built to solve actual business problems.
