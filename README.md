# Customer Support Ticket Bot (n8n + Telegram + Local Gemma 4)

An AI-driven customer support automation built manually in **n8n** that receives user messages via **Telegram**, classifies intent locally using **Gemma 4 (`gemma4:e4b`) via Ollama**, routes tickets with a **Switch** node, fetches live external data via REST API, logs every ticket to **Google Sheets**, and handles edge cases gracefully.

---

## Architecture & Workflow Flow

```text
Telegram Trigger
  └─► Extract & Validate Input (Code)
        └─► Is Valid Message? (If)
              ├─► [False: empty/sticker/photo] ──────────────────────────┐
              └─► [True] ──► Classify Intent - Gemma 4 (Ollama HTTP) ────┼─► [Error: Ollama down/timeout] ──┐
                               └─► Parse & Validate AI JSON (Code)       │                                  │
                                     └─► Route by Category (Switch)      │                                  │
                                           ├─► billing   ──► Google Sheets ──► Telegram Reply               │
                                           ├─► technical ──► Google Sheets ──► Telegram Reply               │
                                           ├─► weather   ──► Open-Meteo API ──► Google Sheets ──► Reply     │
                                           ├─► general   ──► Google Sheets ──► Telegram Reply               │
                                           └─► Fallback  ───────────────────────────────────────────────────┴─► Reply Graceful Fallback (Telegram)



## Links & Storage Details

- **Telegram Bot:** [@sheffield1_bot](https://t.me/sheffield1_bot) *(Note: Hosted locally via Docker, Ollama, and ngrok—active when the local runtime is running)*
- **Logging Destination:** Google Sheets (https://docs.google.com/spreadsheets/d/1hQ7hv45evuEfrhs-2CdpHfQyFc5HHQeR8kbZpDde7j8/edit?usp=sharing)                                           