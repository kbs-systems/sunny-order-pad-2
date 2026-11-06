# Handoff: making Sunny Order Pad a standalone app

**For:** Miles · **From:** Dorian (KBS) · **As of:** Nov 5, 2026 (updated with receiving)

`index.html` is a working app that Jesus, Armando and Gavino use today, but it runs only inside Claude. The goal is the same app running on its own: opened from a link or a home-screen icon on any phone, with no Claude login. Most of the code carries over. Four things that Claude provides today need your own versions.

## 1. Keep as is

Everything in `index.html` except the runtime calls in section 2:

- **Screens and flow.** Catalog by section with search and +/− steppers, Review order (yellow legal-pad layout), Sent orders split into To receive / Filed / All, with status Sent → Ordered → Received, filter by person, Copy text, Order again.
- **Barcode decoding, on the phone.** `decodeBarcode()` (~line 282) uses the browser's `BarcodeDetector` when available, else ZXing (`@zxing/library@0.21.3`). It tries 4 sizes and rotations.
- **Spanish/English.** All UI strings are in the `T` object (~line 187), with `es` as the default. Section and unit names are translated by `catName()` / `unitName()`. Product names are never translated.
- **"Who's ordering?" picker.** `STAFF` (~line 184): Jesus (inventory lead), Armando, Gavino. Remembered per phone.
- **Double-order check.** `recentFor()` (~line 251) flags any item that is in an order from today or the 3 days before. The order sent to Sunny is always English (`orderText()`).
- **Receiving check-off.** `receive()`, `recvSummary()`, `showResult()` (~lines 415–470). See section 3a.
- **Per-phone storage.** `localStorage` keys `sunny-cart` (unsent order), `sunny-lang`, `sunny-who`. These can stay client-side.

## 2. Replace: what Claude provides today

| Today in `index.html` | Where | Replace with |
| --- | --- | --- |
| `claude.use("db")`: `db.collection("items")`, `db.collection("orders")`, live `onSnapshot` | ~lines 340, 366, 385, 408, 465, 529, 546–547 | Your database + API. Live updates are nice-to-have: Sent orders and the double-order check need fresh orders, so poll or subscribe. |
| `claude.use("sample")`: `identify()`, `readList()`, `toEnglish()`, `readReceipt()` | ~lines 295–312, 429 | Server endpoints that call the Claude API (Anthropic) with a server-side key. The prompts in those four functions can be reused as written. They return JSON. |
| `claude.use("user")`: `user.id()`, `user.can("data.write")`, `user.profiles()` | ~lines 495, 543 | Your own login and roles (see open decision 3). |
| No outside network calls | — | UPC database lookup from your server (section 4). |

The startup block (~line 535 onward) is where all three are wired in. If a call is missing, it already falls back gracefully (`sample` null hides the photo features), which is handy while porting.

## 3. Data model

Build for **Sunny Liquor only** for now; it's the pilot. Once Sunny Singh approves it, the same app is set up for **Mark's Liquor**, then **Tres Amigos Market**. All three run NRS POS and buy from the same wholesalers.

- Put a `store_id` on every record from day one, so the next store is setup, not a rebuild.
- Keep wholesalers (vendors) as their own shared list, since all three stores order from the same ones. Each store keeps its own items, people and orders.

**items**: `id`, `name`, `size`, `unit` (Bottle | Can | Pack | Case | Each), `category` (14 fixed values, see `CATS` ~line 181), `price`, `priceDeal` (e.g. "2 for $1"), `barcode`, `active`, `source`, plus `sku`, `subcategory`, `brand`, `vendor`, `cost`, `price`, `par`, `reorderQty` (mostly empty today), `addedBy`, `createdAt`.

- Barcode matching ignores leading zeros (`normCode()`), so UPC-A and EAN-13 forms match.
- `items.csv` in this repo holds all 442 current items, including 12 fresh produce items with prices (added for EBT; `price_deal` column holds multi-buy prices). IDs starting `sl-` map to the store's master inventory record IDs.

**orders**: `id`, `createdAt` (ms), `byLabel` (orderer name), `status` (sent | ordered | received), `note` (English), `noteOriginal` (Spanish original, if translated), `lines[]` of `{itemId, name, size, unit, category, qty}`.

- Lines copy the item's name and size at order time, so old orders stay readable if an item changes.
- Order history isn't in this repo. Dorian can export it when you're ready to import.

Receiving adds to the order: `receiving.pickup` and `receiving.store`, each `{by, byId, at, qty[], note, extras[]}`. `qty[i]` is the count received for `lines[i]`; `extras` lists items on the receipt that weren't ordered. Saving the store check sets `status: "received"` and `issues` (number of lines with a difference).

## 3a. Receiving: the order becomes the check-off list

After an order is sent, the same list is used to check in the merchandise, then the order is filed.

1. **At pickup (optional):** at the wholesaler, while loading the van. Saving it moves a Sent order to Ordered.
2. **At the store:** when unloading, or when a delivery arrives. It pre-fills from the pickup counts. Saving it marks the order Received and files it.
3. **Per line:** +/− counts, or "Everything's here" for all lines. Each line shows All here, N short, Missing or N extra.
4. **Receipt photo:** Claude reads the wholesaler's receipt (`readReceipt()`), maps it to order lines and fills in counts for the person to check. Items on the receipt but not on the order are listed separately.
5. **Result:** three buckets, so a loss can be traced:
    - **Short at the wholesaler:** pickup < ordered, or store < ordered when there was no pickup check.
    - **Lost between pickup and store:** store < pickup.
    - **Extra received:** store > the larger of ordered and pickup.

The result copies as English text for Sunny. Not built yet, and worth doing in the standalone version: an alert to Sunny Singh as soon as a pickup check finds a shortage, while the van is still at the wholesaler.

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
- [ ] An order can be checked off at pickup and at the store, and filed with any shortages listed.
- [ ] The 442 items and the order history are imported, nothing retyped.
- [ ] A second store can be added with its own items and people, without code changes.

**More detail:** the build spec doc "Sunny Order Pad — Barcode Lookup Build Spec" has the lookup flow diagram and the phased plan. Ask Dorian to share it with you.
