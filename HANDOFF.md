# Handoff: making Sunny Order Pad a standalone app

**For:** Miles · **From:** Dorian (KBS) · **As of:** Nov 5, 2026

`index.html` is a working app that Jesus, Armando and Gavino use today, but it runs only inside Claude. The goal is the same app running on its own: opened from a link or a home-screen icon on any phone, with no Claude login. Most of the code carries over. Four things that Claude provides today need your own versions.

## 1. Keep as is

Everything in `index.html` except the runtime calls in section 2:

- **Screens and flow.** Catalog by section with search and +/− steppers, Review order (yellow legal-pad layout), Sent orders with status Sent → Ordered → Received, filter by person, Copy text, Order again.
- **Barcode decoding, on the phone.** `decodeBarcode()` (~line 260) uses the browser's `BarcodeDetector` when available, else ZXing (`@zxing/library@0.21.3`). It tries 4 sizes and rotations.
- **Spanish/English.** All UI strings are in the `T` object (~line 179), with `es` as the default. Section and unit names are translated by `catName()` / `unitName()`. Product names are never translated.
- **"Who's ordering?" picker.** `STAFF` (~line 176): Jesus (inventory lead), Armando, Gavino. Remembered per phone.
- **Double-order check.** `recentFor()` (~line 229) flags any item that is in an order from today or the 3 days before. The order sent to Sunny is always English (`orderText()`).
- **Per-phone storage.** `localStorage` keys `sunny-cart` (unsent order), `sunny-lang`, `sunny-who`. These can stay client-side.

## 2. Replace: what Claude provides today

| Today in `index.html` | Where | Replace with |
| --- | --- | --- |
| `claude.use("db")`: `db.collection("items")`, `db.collection("orders")`, live `onSnapshot` | ~lines 318, 344, 363, 386, 443, 458–459 | Your database + API. Live updates are nice-to-have: Sent orders and the double-order check need fresh orders, so poll or subscribe. |
| `claude.use("sample")`: `identify()`, `readList()`, `toEnglish()` | ~lines 273–290 | Server endpoints that call the Claude API (Anthropic) with a server-side key. The prompts in those three functions can be reused as written. They return JSON. |
| `claude.use("user")`: `user.id()`, `user.can("data.write")`, `user.profiles()` | ~lines 415, 455 | Your own login and roles (see open decision 3). |
| No outside network calls | — | UPC database lookup from your server (section 4). |

The startup block (~line 448 onward) is where all three are wired in. If a call is missing, it already falls back gracefully (`sample` null hides the photo features), which is handy while porting.

## 3. Data model

Put a `store_id` on every record from day one. Four more stores are planned (Mark's, Lee, Deep, #Amigos Market), all on NRS POS.

**items**: `id`, `name`, `size`, `unit` (Bottle | Can | Pack | Case | Each), `category` (13 fixed values, see `CATS` ~line 173), `barcode`, `active`, `source`, plus `sku`, `subcategory`, `brand`, `vendor`, `cost`, `price`, `par`, `reorderQty` (mostly empty today), `addedBy`, `createdAt`.

- Barcode matching ignores leading zeros (`normCode()`), so UPC-A and EAN-13 forms match.
- `items.csv` in this repo holds all 430 current items. IDs starting `sl-` map to the store's master inventory record IDs.

**orders**: `id`, `createdAt` (ms), `byLabel` (orderer name), `status` (sent | ordered | received), `note` (English), `noteOriginal` (Spanish original, if translated), `lines[]` of `{itemId, name, size, unit, category, qty}`.

- Lines copy the item's name and size at order time, so old orders stay readable if an item changes.
- Order history isn't in this repo. Dorian can export it when you're ready to import.

## 4. New: online barcode lookup

When a scanned barcode isn't in `items`, check in this order:

1. The store's own item list.
2. A UPC database, called from the server.
3. Claude reading the label photo.
4. Manual entry.

Save whatever answers with the barcode, so each product is looked up once.

- **UPCitemdb** has a free tier of 100 lookups/day, which is likely enough for one store because of the look-up-once rule.
- **Backups:** Barcode Spider (~$9/mo) and Go-UPC (~$19.95/mo). Pricing is from a third-party comparison; confirm on each vendor's site.
- **Test coverage first.** No service clearly states its liquor, beer or Mexican-snack coverage. Before choosing, test 20 real shelf barcodes: 6 spirits, 6 beer, 4 Mexican snacks, 4 grocery.

## 5. Suggested architecture

- **Phone app:** installable web app (PWA), same screens as `index.html`.
- **Small server:** UPC lookup, Claude API calls, order alerts.
- **Database:** one, shared by all stores.
- **Server-side secrets:** `ANTHROPIC_API_KEY`, `UPC_API_KEY`, `GHL_WEBHOOK_URL`. Never put them in the phone app.
- **KBS monitoring:** a one-way webhook to GoHighLevel (`GHL_WEBHOOK_URL`) for order events. The app must keep working if it fails.
- **NRS POS:** there is no public API. Sync is by CSV: import NRS item and sales exports, and export back only if NRS allows bulk import. An NRS item export, when it arrives, fills in barcodes and prices.

## 6. Open decisions (Dorian)

- [ ] Who owns the hosting, database and API accounts: KBS or the store group?
- [ ] Is the app free to the stores during the pilot, or part of a KBS tier?
- [ ] Login: a PIN per person, or phone-number sign-in?
- [ ] Does NRS allow bulk import, so price edits can flow back to the POS?

## 7. Done means

- [ ] Opens on Jesus's phone from a home-screen icon, with no Claude account.
- [ ] Everything in section 1 works as it does today, in Spanish by default.
- [ ] An unknown barcode fills in name and size without typing, then scans instantly next time.
- [ ] Sunny Singh sees new orders and can mark them Ordered and Received.
- [ ] The 430 items and the order history are imported, nothing retyped.
- [ ] A second store can be added with its own items and people, without code changes.

**More detail:** the build spec doc "Sunny Order Pad — Barcode Lookup Build Spec" has the lookup flow diagram and the phased plan. Ask Dorian to share it with you.
