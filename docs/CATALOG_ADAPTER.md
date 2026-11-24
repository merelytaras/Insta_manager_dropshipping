# Catalog adapter (workflow "Catalog - Sync (adapter)")

> Status (2026-11-24): PARKED by the user. Built as workflow `YjmbUdQxFCKSy8kG` (unpublished, writes to test table `catalog_test`). Four test runs all failed or were killed after 6-10 min; not verified. Resume when a real client sends a feed. The old sync (`Nl6uEgA2gmiu6SAf`) is the working one.
> Workflow state left as-is on purpose: it may still contain the TEMP node `Feed size` and the two save nodes may be disabled (diagnostics); clean up before resuming.

## Test findings (2026-11-24)
- Demo feed is only 5.3 MB / 10,153 `<item>` (about 923 models x ~11 sizes); download takes < 2 s.
- Runs held ~324 MB of execution data (n8n: 'too large to display'), which also broke the n8n MCP connection when reading them.
- With the save nodes disabled the run was still going after 5+ min; the UI showed a data-table icon on the running step, so the likely bottleneck is `Get existing catalog` (returnAll on the 923-row table), not download/XML/saves. Not proven.
- Fix ideas: keep a hash snapshot (group_id -> hash) in ONE settings row and diff against it instead of reading all rows (update the snapshot only after saves succeed); or merge download+parse+diff into one Code node so only changed rows are passed on.
- Lessons: pause the old sync before testing (overlap stalled polling for ~12 min); a docker restart kills running executions ('crashed'); the n8n MCP drops when a run is heavy.
- The old sync is currently UNPUBLISHED (by the user, for the test): the catalog is not refreshing until it is republished.

Goal: plug any client's product data into the bot in about an hour. One workflow downloads a feed from a URL, detects its format, normalizes it, and writes **only changed rows** into the catalog data table. Replaces the old per-client "Sync catalog" workflows.

## Configuration
- n8n settings data table (`key` / `value`): row `catalog_url` = feed URL (may contain a secret key; it lives only in n8n, never in git).
- Catalog destination: the catalog data table. The workflow's save nodes point to it (during testing they pointed to `catalog_test`).

## Supported formats (auto-detected)
| Format | How detected | Notes |
|---|---|---|
| Prom XML (`<shop><items><item>`) | starts with `<`, has `items/item` | what EasyDrop / Prom export gives. One `<item>` = one size variant; grouped by `group_id` |
| YML (`<yml_catalog><shop><offers><offer>`) | starts with `<`, has `offers/offer` | Horoshop, Rozetka, many CMS. `group_id` attribute groups sizes; size comes from `<param name="Розмір|Размер|Size">`; stock from `stock_quantity`/`quantity`, else `available="true"` = 1 |
| CSV / Google Sheet | anything not starting with `<` | publish the sheet as CSV (File → Share → Publish to web → CSV). `,` or `;` delimiter auto-detected |

### CSV columns (header names are case-insensitive; synonyms accepted)
- model: `model`, `name`, `назва`, `модель`, `название` (required)
- `group_id` / `group` (optional; rows with the same group = sizes of one model; default: same model name)
- size: `size`, `розмір`, `размер`
- stock: `qty`, `quantity`, `stock`, `залишок`, `кількість`, `остаток` (missing column = every row in stock)
- price: `price`, `ціна`, `цена`
- photo: `image`, `photo`, `фото`, `picture`, `image_url` (several URLs separated by space, `;` or `|`)
- `code` / `sku` / `артикул`, `description` / `опис`
- any other column becomes a "param" shown to the bot (brand, material, season…)

## Normalized row (one per model) written to the catalog table
`model, group_id, price, sizes_in_stock ("42 (3), 43 (1)"), sizes_out, in_stock_count, codes, image, images, params, description`

## Safety guards
- Aborts (error → alert) if the feed is empty, unparseable, produces < 10 models, or shrinks to < 50% of the current catalog.
- Writes only new/changed models (the old sync rewrote all ~923 rows every run and took 7–10 minutes).
- Models that disappear from the feed are marked out of stock (`sizes_in_stock=''`, `in_stock_count=0`), never deleted.

## Not covered yet
- JSON APIs (e.g. EasyDrop `/supplier-items/` + `/supplier-sizes/`): needs a dedicated fetch step that merges items and sizes into variants, then the same normalization.
- Authenticated feeds (header token): add a header-auth credential on the Download node.
- Several warehouses: currently the feed's single quantity field is used; summing warehouses is a per-client decision.
