# Sunny Order Pad

A phone-first ordering tool for Sunny Liquor. The people who order stock (Jesus, Armando and Gavino) build an order by tapping items, scanning a barcode, snapping a product photo, or snapping a handwritten list, then send it to the owner, Sunny Singh. Built by Keystone Business Systems (KBS).

## What's in this repo

| File | What it is |
| --- | --- |
| `index.html` | The whole app in one file: markup, styles and script. |
| `items.csv` | The store's item list as of 11/04/2026: 430 products. |
| `README.md` | This file. |

## Important: it currently runs inside Claude

This version was built as a Claude artifact. It depends on Claude's page runtime (`window.claude`) for:

- **Database (`db`)**: the shared item list (`items` collection) and sent orders (`orders` collection).
- **Ask Claude (`sample`)**: reads product labels, barcodes and handwritten lists from photos, and translates Spanish notes to English.
- **Viewer identity (`user`)**: who is signed in and whether they may write.

Opened anywhere else (GitHub Pages, a local file), the page loads but shows "Open this page in Claude" and has no data. **The job:** turn it into a standalone app by replacing those three pieces with a backend (e.g. Supabase/Firebase or a small Node server + Postgres), auth for the three orderers plus Sunny, and direct vision/LLM API calls for photo reading and translation. `items.csv` seeds the `items` table.

Barcode decoding does not need Claude: it uses the browser's `BarcodeDetector` when available, falling back to ZXing (`@zxing/library@0.21.3` from jsDelivr).

## Features to keep

- Spanish by default, with an ES/EN switch remembered per phone. Product names stay as printed on the label; orders sent to Sunny are always in English (Spanish notes are translated, original kept).
- "Who's ordering?" picker (Jesus as inventory lead, Armando, Gavino), remembered per phone. Every order and new item records who made it.
- Catalog by section with search, tap-to-add and +/− steppers. Unknown barcodes or photos are identified and saved to the list with their barcode.
- Double-order check: warns when an item was already ordered today or in the 3 days before.
- Sent orders with status Sent → Ordered → Received, a filter by person, Copy text, and Order again.
- Receiving: check the order off at pickup and/or at the store (counts by hand or from a receipt photo), see what was short, lost or extra, and file it.

## Data model

**`items/{id}`**

| Field | Notes |
| --- | --- |
| `name`, `size`, `unit` | `unit` is one of Bottle, Can, Pack, Case, Each |
| `category` | Spirits, Beer & Seltzer, Drinks, Snacks, Candy & Gum, Grocery, Refrigerated & Desserts, Household & Cleaning, Paper & Supplies, Personal Care, Pet, Automotive, Other |
| `barcode` | UPC/EAN digits; matched with leading zeros ignored |
| `sku`, `subcategory`, `brand`, `vendor`, `cost`, `price`, `par`, `reorderQty` | From the master inventory list where available; mostly blank today |
| `active` | `false` hides the item from the order screen |
| `source` | Where the item came from (master list, handwritten list, shelf photo) |

**`orders/{id}`**: `createdAt` (ms), `byLabel` (orderer name), `status` (sent, ordered, received), `note` (English), `noteOriginal` (Spanish original, if translated), `lines[]` of `{itemId, name, size, unit, category, qty}`, and after receiving `receiving.{pickup,store}` and `issues`.

## items.csv notes

- `id` matches the item's database id. SKUs starting `SL-` are the store's master inventory record IDs.
- Barcodes, costs and prices are empty for now; they're expected from an NRS POS item export.
- The store's POS (NRS) has no public API; plan for a periodic CSV import.
