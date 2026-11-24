# Catalog adapter (workflow "Catalog - Sync (adapter)")

> Status (2026-11-24): built as workflow `YjmbUdQxFCKSy8kG`, UNPUBLISHED, writing to test table `catalog_test`; first test run against the live feed was still in progress when this was written. Not yet verified or switched over from the old sync (`Nl6uEgA2gmiu6SAf`).

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
