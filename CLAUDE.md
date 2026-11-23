# CLAUDE.md — read this first

This repo is the long-term memory for Taras's Instagram/Telegram shoe-shop chatbot (n8n). Chat context gets compacted, so:

1. At the start of every session read `docs/NOTES.md` (full state, decisions, what worked/failed, TODOs) and `README.md`.
2. After every significant change (workflow, decision, test result, client news) update `docs/NOTES.md` and commit + push. Keep a short changelog line in `docs/CHANGELOG.md`.
3. Never commit secrets: API keys, tokens, passwords, full credential values, customers' personal data. Workflow exports in `workflows/` must have credentials stripped.
4. The repo is currently PUBLIC. Keep client names/IDs minimal and suggest making it private.
5. The user (Taras) writes in English or Ukrainian; bot replies to customers are in Ukrainian.
