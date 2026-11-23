# MASTER NOTES — Insta Manager for Dropshipping

> Read FIRST after any context compaction. Last updated: 2026-11-23.
> No secrets here. Credentials live in n8n only. Repo is public: keep client details minimal.

## 1. Who / goal
- **Taras**, solo freelancer in Ukraine (Europe/Kyiv), learning n8n, wants to earn money from automations.
- **Product:** AI chatbot for **shoe dropshipping shops**, answering in Instagram DMs and Telegram: identifies a model from a customer photo, checks stock and sizes, offers alternatives, collects the order, hands off to a human only when really needed.
- **Two variants:**
  - shops with their own catalog → workflows named **"Shoes - …"** (built for client Rik Shoes);
  - dropshippers on **EasyDrop PRO** → **"EasyDrop demo - …"** copies (demo client `shmotkin_lab`).
- **LLM/embedding models are configurable per client** (now: OpenAI `gpt-6.1-terra` + Gemini `gemini-embedding-2`; not a requirement).
- **Pricing plan:** $300 setup, $25/month, 5-day trial.
- **Current deal:** `shmotkin_lab`. Demo runs on test Instagram page `spreadingluck` with the EasyDrop workflow copies, tested from Taras's own IG accounts.
- **Rik Shoes** polling workflow `tSgSpTdr0ANH3dkO` is **UNPUBLISHED during the demo** and MUST be republished when Taras says so. An "EDR dispatcher" was added to the Rik approval workflow `m9HYqSX0eLpCeQNk` (remove after the demo).

## 2. Standing product decisions (do not re-ask)
- Bot never replies in Russian unless the customer asks. Ukrainian by default.
- Greets only on a bare hello.
- No prepayment.
- FAQ allowed, but info must come from the shop owner (never invented).
- Follow-up nudge: yes. Config cloneable per client: yes.
- Identifies models from photos, shows ONE photo and asks "Це ця модель?", checks stock/sizes, offers other sizes/colours/similar models, honest when something isn't in the catalog.
- Web-sourced claims: "за описом бренду для цієї моделі" + "виріб може відрізнятися"; NEVER "постачальник повідомляє" (catalog prices and empty manufacturer suggest non-original goods, unconfirmed).
- `drop` / `drop_price` (supplier cost) must NEVER be stored for, or shown to, customers. Retail price comes from the XML catalog.

## 3. Infrastructure
- **n8n runs locally in Docker** on Taras's laptop (must stay on and awake). **ngrok** exposes it for Telegram (URL never changed so far). Production needs an always-on server before selling the $25/month plan.
- n8n is driven by Claude through MCP server `Local_n8n_mine` (sometimes disconnects; retry).
- The agent model prioritises tool descriptions and often ignores prompt rules → **server-side guards are more reliable than prompt rules**.
- Photo search: embeddings cosine similarity + vision compare.
- n8n credentials (names only): Instagram token, Telegram, Gemini, OpenAI, header-auth `Smotkin_EasyDrop` (value must be `Token <key>`, key NOT yet received).

## 4. Workflows (n8n IDs)
| ID | Name / role | State |
|---|---|---|
| `1P0iHc1pnokEWwAE` | EasyDrop demo – Conversation handler (agent, prompt, tool descriptions) | live |
| `f4U8TRP0hX53kiSC` | EasyDrop demo – Instagram polling (`/conversations` every minute) | live |
| `MqQcE8lKX3GE11Me` / `LC3Y0X8tV3BpFhZM` / `h4Fz42yuyODS13Gf` | EasyDrop demo – Tool create order / Approve order / Assistant (chat) | published / published / unpublished |
| `NucsDBVgwqLWLGIF` | Shoes – Conversation handler (Rik Shoes) | unpublished (demo) |
| `1TtwHeKw1ccCbxCG` / `xA4fcf6Dcf392R6W` / `Gq5aJ3cNGvW1Iloj` | Shoes – Tool create order / Owner commands / Refresh Instagram token (archived) | unpublished |
| `31ovH3Px6lDIwwHI` | EasyDrop demo – Tool: check stock (alias table, winter search) | published |
| `Vlq1mngvArKqad7o` | Shoes – Tool: handoff to human (+ guard), shared with Rik | published |
| `Jz1HNEjcAJaIJZwp` | EasyDrop demo – Follow-up nudge | published |
| `pqBvLMUs6iuQ2n4G` | Send photo | live |
| `hozU2zy48wmm9bRJ` | Photo index sync | live |
| `Nl6uEgA2gmiu6SAf` | Sync catalog from Prom XML (every 15 min) | live |
| `VbgYfCud2YasU2oB` | Weekly report | live |
| `4zkNjBAi75zHJnjQ` | Webhook probe | live |
| `bAz0D5BejaM5Qz7i` | Find by photo | live |
| `717JIPmKT6PouevB` | Shoes – Error alert | UNPUBLISHED |
| `tSgSpTdr0ANH3dkO` | Shoes – Instagram polling (Rik Shoes) | UNPUBLISHED (republish after demo) |
| `m9HYqSX0eLpCeQNk` | Shoes – Owner approval (Telegram buttons); has EDR dispatcher to remove later | published |

Full JSON exports (secrets/PII stripped) are in `workflows/` (see `workflows/README.md`). Other n8n workflows on the laptop (YouTube analytics etc.) belong to a different project and are not in this repo.
| `PdjcluvRuvblWUFS` | TEMP test harness | unpublished, archive later |

## 5. Data tables (n8n project `kP67zQab1RNBdvlR`)
- `73uDrVHYxs5M5e5V` catalog (923 rows: model, group_id, price, sizes_in_stock, sizes_out, in_stock_count, codes, image, images, params, description "Матеріал : ... Виробник : -")
- `YV1e41fJ7rUEv9bt` photo index · `2B06l4Xq0NTtxFdg` chat log · `e5CQ1qnS6daMk9Kq` orders · `JbkN3T4Qea93RNPY` handoff pause · `jQTafqjKvGXLfhuJ` settings · `t7PkfUtoPYsNYuu5` customers · `ujZf1NCiXjCv42q1` webhook log

## 6. What we built and how it works
- **Catalog** comes from a Prom XML export synced every 15 min. Probably incomplete (no TNF / Columbia), weak descriptions, no season field, some missing photos.
- **check_stock** (`31ov…`, node "Find model and sizes"): alias table for slang/brands (нбешки, адіки, пуми, найки, тнф, угг, афки…; "чобіт/черевик/ботінк/сапог"→boot), stop-word list, tiered matching r1–r4. Winter intent (зима, тепл, утеплений, дощ, водостійн…) filters by name/material keywords: Fur/boots = insulated, Gore-Tex = waterproof, Cordura = durable textile (not necessarily warm). Returns up to 8 models with `model_id`, price, sizes. Verified on real catalog: "що є на зиму", "найки на зиму 42", "чоботи 38", "водостійні кросівки 43".
- **Handoff guard** (`Vlq1mngvArKqad7o`): IF node "Handoff allowed?" blocks handoffs whose reason is a data gap (утепл/сезон/матеріал/немає даних/не знайдено…) unless a real reason is present (client asks for human, complaint, order problem, delivery, payment, return, warranty). Blocked branch tells the agent to call check_stock, suggest 2–3 models, use web search, ask gender/size/brand/budget, and not mention handoff. **Not yet verified live.**
- **Context-aware nudge** (`Jz1HNEjcAJaIJZwp`): window 3–20 h after last message, 10:00–21:00 Kyiv, max 5 per run, skips customers with orders or handoff pause. Text depends on last agent reply: order summary pending → reminds item, asks to confirm; size question pending → asks size for chosen model; else generic. Default nudge confirmed live; context variants tested only in Node.

## 7. What worked / what didn't (and why)
**Worked:** photo → catalog match; stock/size answers; slang aliases; winter search; default nudge live; the bot reads story replies (confirmed on Rik Shoes).

**Didn't work / limits**
- **Instagram shared posts, reels, voice notes, phone cards** arrive as EMPTY messages via polling. Webhooks would carry them (only a URL for shares) but **no webhook events are delivered**: webhooks reach an app only for people with a role on it unless the app passed App Review and is Live. Swipe-reply target not visible. Image attachment can't carry a caption. 24 h reply window; ~1000 char limit.
- **ManyChat does NOT fix this:** cannot trigger on posts/reels shared by DM (moderator confirmed); `instagram_attachment_url` often empty; External Request times out ~10 s but our agent takes up to ~20 s. Its advantages are comment-to-DM, story triggers, flow builder, tags, live chat. (Earlier claim that story replies were a ManyChat advantage was wrong.)
- **"що є на зиму" caused handoff:** agent searched "зима"/"хутро" (empty / returned all 923 because "хутро" was a stop word), then handed off with "немає даних про утеплення". Fixed with winter-intent search + handoff guard.
- **n8n MCP gotchas:** `setNodeParameter` with path `/parameters/jsCode` creates a nested object → use `updateNodeParameters` with `replace:true` and full parameters (jsCode, language, mode). Sub-workflows run the PUBLISHED version → `publish_workflow` before testing. `execute_workflow` needs `executionMode:'manual'` and several polls. Foreground `sleep` is blocked (use Monitor). `get_data_table_rows` needs `projectId`. Conversation handler can't be executed via MCP, needs live tests.
- **Catalog gaps:** no TNF/Columbia, no season/material/technology fields. "водостійке з хутром" = no data; root cause is catalog, not logic.

## 8. EasyDrop API (planned fix for catalog gaps)
- Base `https://easydrop.one/api/v1`, DRF token auth (`Authorization: Token <key>`).
- GET `/supplier/`, `/supplier-items/` (supplier_id, id, limit default 1000; fields id, model, brand, code, manufacturer, material, drop, link, drop_price, nal, image), `/supplier-sizes/` (supplier_id; id, item_id, item_name, item_code, val, sock, qty), `/supplier-item-images/` (item_id required).
- No retail price in API (only drop prices); retail price from the XML.
- **Status:** owner said he'd look; Taras sent a Google Doc with proof/docs. Key NOT received. Must be passed safely (not via IG DM; never send the n8n credentials URL to the owner).
- **When the key arrives:** Taras pastes it into credential `Smotkin_EasyDrop` as `Token <key>` (Claude must not see it). Then test `/supplier/`, `/supplier-items/`, `/supplier-sizes/`; compare with the 923 models; look for TNF; check material/manufacturer data; add daily key-validity check (enable `Shoes - Error alert` `717JIPmKT6PouevB`); never expose drop prices.
- **Plan B (no API / too little data):** enrichment pipeline — table `catalog_enriched` keyed by group_id, workflow "Enrich new models" after Sync catalog, photo + web search, confidence/source fields; pilot ~20 models, then backfill in-stock first; update check_stock to use enriched fields.

## 9. Meta app review (selling to many clients)
- Standard Access: only people with a role on the app. Advanced Access needs App Review; required for accounts you don't own. Business Verification needed when users have no role on the app. App must be Live.
- Review is per app, not per client, but each client must authorize the app themselves (OAuth / Facebook Login for Business; IG account linked to a Page). Can't "swap accounts". Docs don't explicitly say one approval covers all clients — verify before promising.
- Review needs working demo, screencast, privacy policy; reviewers must be able to test.
- Build a "Connect Instagram" login flow; until then add early clients as testers.
- Sources: developers.facebook.com/docs/instagram-platform/overview, /docs/instagram-platform/app-review.

### Meta App Review progress (2026-11-23)
- Started a submission (NOT submitted yet). Trimmed to 2 permissions: `instagram_basic`, `instagram_manage_messages` (removed ads_*, content_publish, insights, events; skipped comments and Human Agent for now).
- App is Published (Live). Sidebar shows Facebook Login for Business (clients need IG linked to a Facebook Page).
- Privacy policy + data deletion: public Google Doc (published to web, auto-republish). Promises 90-day retention of conversations → TODO: n8n cleanup workflow deleting chat logs older than 90 days. Policy lists n8n, OpenAI, Gemini, Telegram as processors; keep it in sync with reality.
- Still needed before submit: use-case text + step-by-step (drafted in chat), screen recording of DM → bot reply, test customer Instagram account for reviewers, check Business Verification (Taras has no registered business; maybe sole-proprietor docs).
- Meta now reviews the WHOLE app on each submission and submissions can't be edited/cancelled once in review.

## 10. Open TODOs
1. Wait for shmotkin_lab owner: API key, whether TNF is in EasyDrop, API tariff.
2. Live retests by Taras (press "Повернути бота" in Telegram first): "вітаю, що є на зиму?", "кросівки на зиму 42", "чоботи 38", "New Balance 9060 Cordura на зиму чи ні?", "нбешки 42", "адіки білі 38", "афки чорні 40". Confirm handoff guard and new nudge texts (order-summary nudge after ~3 h).
3. FAQ block (owner must answer: delivery cost/time, try-on, return/exchange, warranty, payment methods, working hours/manager reply time).
4. Refine "brand not in catalog" message (e.g. TNF).
5. Voice/shared posts via webhooks (needs Meta app approval; Facebook Trigger as plan B).
6. After the demo: republish Rik Shoes `tSgSpTdr0ANH3dkO`; optionally unpublish EasyDrop polling and nudge `Jz1HNEjcAJaIJZwp`; remove EDR dispatcher in `m9HYqSX0eLpCeQNk`; archive harness `PdjcluvRuvblWUFS`; decide when to republish `Shoes - Error alert`.
7. Move n8n to an always-on server before selling the $25/month plan; keep laptop on and awake during the demo.
8. Make this repo private (it was created public).

## 11. Timeline (latest chat segment)
alias check → TNF on shmotkin page, API idea → how to pass/verify key → message to owner (sent) → owner "will look" → n8n credential created → reassurance message + proof doc → winter handoff bug fixed → context-aware nudge → ManyChat research → Meta app review question → repo created (2026-11-23).
