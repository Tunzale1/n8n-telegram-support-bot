# Customer Support Ticket Bot (n8n + Telegram + Local Gemma 4)

An AI-driven customer support automation built manually in **n8n** that receives user messages via **Telegram**, classifies intent and language locally using **Gemma 4 (`gemma4:e4b`) via Ollama**, routes tickets with a **Switch** node, fetches live external weather data via REST API, logs every ticket to **Google Sheets**, and handles rate limits, retries, admin alerts, and edge cases gracefully.

---

## Links & Storage Details

- **Telegram Bot:** [@sheffield1_bot](https://t.me/sheffield1_bot) *(Hosted locally via Docker, Ollama, and ngrok—active when the local runtime is running)*
- **Logging Destination:** [Google Sheets — `Support_Tickets` (`Tickets` worksheet)](https://docs.google.com/spreadsheets/d/1hQ7hv45evuEfrhs-2CdpHfQyFc5HHQeR8kbZpDde7j8/edit?usp=sharing)
- **Workflow Export:** [`customer-support-ticket-bot.json`](customer-support-ticket-bot.json)

---

## Architecture & Workflow Flow

```text
Telegram Trigger
  └─► Extract & Validate Input (Code: metadata, rate-limit check, /history detection)
        └─► Intake Guard & Commands (Switch)
              ├─► [Rate Limited (>5/min)] ──► Reply Rate Limit (Telegram)
              ├─► [History Command] ────────► Fetch Ticket History (Sheets) ──► Format Last 5 Tickets (Code) ──► Reply History (Telegram)
              ├─► [Invalid Input] ──────────┼──────────────────────────────────────────────────────────────────────────────────────────┐
              └─► [Valid Ticket] ───────────► Classify Intent - Gemma 4 (Ollama HTTP) ──► [Error: Ollama down/timeout] ────────────────┤
                                                └─► Parse & Validate AI JSON (Code)                                                    │
                                                      └─► Needs Retry? (If)                                                            │
                                                            ├─► [True] ──► Retry Classify - Strict Prompt ──► Parse Retry JSON ───┐    │
                                                            │                └─► [Error] ─────────────────────────────────────────┼────┤
                                                            └─► [False] ──────────────────────────────────────────────────────────┴─┐  │
                                                                                                                                    ▼  │
                                                                                                          Route by Category (Switch)   │
                                                                                                            ├─► billing   ──► Log Billing Ticket ──► Reply Billing (az/en/ru)
                                                                                                            ├─► technical ──► Log Technical Ticket ──► Reply Technical (az/en/ru)
                                                                                                            │                                      └──► Notify Admin - Technical
                                                                                                            ├─► weather   ──► Fetch Live Weather ──► Log Weather Ticket ──► Reply Weather
                                                                                                            ├─► general   ──► Log General Ticket ──► Reply General (az/en/ru)
                                                                                                            └─► Fallback  ───────────────────────────────────────────────┴─► Reply Graceful Fallback
```

---

## Core Features Implemented

- **Telegram Webhook Integration:** Configured `@sheffield1_bot` via `@BotFather` and connected it to n8n's `Telegram Trigger` node over HTTPS (`ngrok`).
- **Local AI Classification (Zero Cloud API Keys):** Calls `POST http://host.docker.internal:11434/api/chat` running `gemma4:e4b` in Ollama with `stream: false`, `think: false`, and strict JSON Schema output (`billing | technical | weather | general | unknown`).
- **Intent-Based Switch Routing:** Evaluates the AI-assigned `category` field (without hardcoded keyword matching) across 4 active category branches plus a `Fallback` branch.
- **Google Sheets Persistence:** Appends structured ticket records (`timestamp`, `user_id`, `username`, `full_name`, `category`, `original_message`, `api_result`) to the `Support_Tickets` spreadsheet (`Tickets` tab).
- **Live External REST API:** The `weather` branch queries the public **Open-Meteo API** (`https://api.open-meteo.com/v1/forecast?latitude=40.4093&longitude=49.8671&current_weather=true`) and injects live Baku temperature and wind speed into both the Telegram reply and Google Sheet log.
- **Comprehensive Error & Edge-Case Handling:** Gracefully handles empty/non-text messages (stickers/photos), ambiguous/unclassifiable text, malformed JSON, Ollama downtime/timeouts, and external API errors without crashing.
- **On-Canvas Developer Documentation:** Annotated directly on the n8n workflow canvas using 5 color-coded Sticky Notes.

---

## Bonus Features Implemented

1. **Per-User Rate Limiter (5 msgs/min):** Tracks sliding 60-second request timestamps per `user_id` via n8n global workflow static data (`$getWorkflowStaticData('global')`) and blocks spam with a polite cooldown message.
2. **Admin Notification on Technical Tickets:** Fires a parallel `Notify Admin - Technical` Telegram alert containing full user metadata (`full_name`, `username`, `user_id`, `timestamp`) and the reported issue whenever a `technical` ticket arrives.
3. **Conversation History (`/history`):** Queries Google Sheets for the requesting `user_id` and returns their 5 most recent support tickets formatted in reverse chronological order.
4. **Multilingual Detection & Replies (`az` / `en` / `ru`):** Gemma 4 outputs a structured `language` field alongside `category`, dynamically tailoring customer replies in Azerbaijani, English, or Russian.
5. **Confidence / Strict-Prompt Retry:** If the initial classification returns `unknown` or malformed JSON, `Needs Retry?` routes the message once through `Retry Classify - Strict Prompt` (`temperature: 0.0`) before falling back to `Reply Graceful Fallback`.
6. **Hardware-Aware Model Selection (`gemma4:e4b` on 16 GB RAM):** Selected `gemma4:e4b` (~6.6 GB–9.5 GB footprint) to balance strong multilingual classification accuracy with safe memory headroom while running Windows, Docker, and Ollama concurrently on a 16 GB RAM laptop.

---

## Google Sheets Schema (`Support_Tickets` → `Tickets`)

| Column | Field Name | Description |
| :--- | :--- | :--- |
| **A** | `timestamp` | ISO 8601 UTC timestamp of the incoming Telegram message |
| **B** | `user_id` | Telegram numeric sender ID |
| **C** | `username` | Telegram `@username` (or `no_username` if unset) |
| **D** | `full_name` | Sender's first and last name |
| **E** | `category` | AI-classified intent (`billing`, `technical`, `weather`, `general`) |
| **F** | `original_message` | Raw text sent by the user |
| **G** | `api_result` | Live Open-Meteo weather metrics for `weather` tickets, or `N/A` |

---

## Tech Stack

| Component | Technology |
| :--- | :--- |
| **Workflow Orchestration** | n8n (Self-hosted via Docker) |
| **Local LLM Runtime** | Ollama (`gemma4:e4b`, structured JSON Schema output) |
| **Messaging Interface** | Telegram Bot API + ngrok HTTPS tunnel |
| **Data Storage & History** | Google Sheets (`Support_Tickets`) |
| **External Weather API** | Open-Meteo Forecast REST API |

---

## How to Run Locally

1. **Pull and start Gemma 4 in Ollama:**
   ```bash
   ollama pull gemma4:e4b
   ```
2. **Start an HTTPS tunnel on port `5678`:**
   ```bash
   ngrok http 5678
   ```
3. **Launch n8n in Docker with your ngrok `WEBHOOK_URL`:**
   ```bash
   docker run -d --name n8n -p 5678:5678 \
     -e WEBHOOK_URL="https://<your-ngrok-domain>.ngrok-free.dev/" \
     -v n8n_data:/home/node/.n8n \
     --add-host=host.docker.internal:host-gateway \
     n8nio/n8n
   ```
4. **Import & Activate:**
   - Import `customer-support-ticket-bot.json` into n8n.
   - Attach your **Telegram Bot API** credential and **Google Sheets OAuth2** credential.
   - Toggle the workflow to **Published** and message `@sheffield1_bot` on Telegram.

---