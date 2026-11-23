# Insta Manager for Dropshipping shops

AI chatbot for **shoe dropshipping shops** that answers customers in **Instagram DMs and Telegram**, built on **n8n** by a solo freelancer (Taras).

## What the bot does
- Understands a customer's photo, finds the model in the shop catalog (embedding search + vision compare), shows ONE photo and asks "Це ця модель?".
- Checks stock and sizes, offers other sizes/colours/similar models, is honest when something isn't in the catalog.
- Understands slang/brand aliases (нбешки, адіки, тнф…) and season intent ("що є на зиму").
- Collects the order, hands off to a human only for real reasons (guarded server-side).
- Follow-up nudges for customers who went quiet (context-aware).
- Weekly report, error alerts.

## Two variants
- **Shops with their own catalog** → the "Shoes - …" workflows (built for the Rik Shoes client).
- **Dropshippers** (supplier platform EasyDrop PRO, catalog from Prom XML + EasyDrop API) → the "EasyDrop demo - …" copies (demo client: shmotkin_lab).

The LLM and embedding models are **configurable per client** (currently OpenAI + Gemini, but not a requirement).

## Business
$300 setup, $25/month, 5-day free trial.

## Repo layout
- `docs/NOTES.md` — everything: goal, architecture, decisions, what worked/failed and why, TODOs.
- `docs/CHANGELOG.md` — dated log of changes.
- `workflows/` — n8n workflow exports (credentials stripped).
- `CLAUDE.md` — instructions for Claude to reload context after a compaction.
