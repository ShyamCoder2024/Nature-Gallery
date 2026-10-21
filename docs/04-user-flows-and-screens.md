# 04 — User Flows & Screens

This document describes every key flow step by step and every screen with its states. Screen IDs (`G-xx`, `A-xx`, `K-xx`, `P-xx`) are used in design files and tickets.

---

## 1. Core guest flows

### F-G1 — First-time guest: scan → order → pay at counter

```
Scan QR
  └▶ G-01 Loading shell (branded skeleton, < 1.5 s)
       └▶ G-02 Menu
            ├─ tap item with options ─▶ G-03 Item sheet ─▶ choose variant/modifiers ─▶ "Add · ₹X"
            ├─ tap "+" on simple item ─▶ added (fly-to-cart animation)
            └─ Cart bar "View cart" ─▶ G-04 Cart
                 └─ "Place order"
                      ├─ identity required & unknown ─▶ G-05 Your details (name + phone)
                      │     └▶ G-06 OTP ─▶ verified
                      ├─ cart validation fails ─▶ G-04 with change notice (must confirm)
                      └▶ sending (button progress ring)
                           └▶ G-07 Order status (Sent → Confirmed → Preparing → Ready → Served)
                                ├─ "Add more items" ─▶ G-02 (next order joins same session)
                                ├─ "Your table" ─▶ G-08 Session & bill
                                └─ "Request bill" / "Pay at counter" ─▶ G-08 shows "Bill requested"
                                     └─ staff settles ─▶ G-09 Paid & receipt ─▶ G-10 Feedback (P1)
```

**Step details**

1. **Scan.** Phone camera opens `https://<short-domain>/q/<token>`. The edge resolves the token to `outlet + spot` (doc 07 §4.1) and serves the HTML shell with the theme and the menu snapshot reference.
2. **Menu.** Rendered from the snapshot (server-rendered HTML for first paint, then hydrated). The device gets an anonymous `guest_device` cookie.
3. **Add items.** Cart lives on the device (IndexedDB) and is mirrored to the server in the background (doc 08 §6) so staff could see it if needed (P2 shared cart builds on this).
4. **Place order.** Client generates an idempotency key, validates the cart with the server (`POST /cart/validate`), then sends `POST /orders` (doc 07 §4.4). If offline, the order goes into the device outbox and the UI shows "Sending…".
5. **Identity.** Shown only when the tenant requires it and the device isn't recognised. The cart is preserved and placing continues right after verification (no second tap needed).
6. **Status.** Realtime subscription to the order and session; falls back to polling every 5 s (then 15 s after 2 min) if the socket can't connect.
7. **Bill.** Server-calculated bill in G-08. "Pay at counter" creates a bill request and a service request for staff.
8. **Settlement.** Staff record the payment and settle; guest screen changes to G-09 automatically.

### F-G2 — Returning recognised guest

1. Scan → menu shows "Welcome back, Riya" and a **Your usual** row (top 6 items by frequency at this tenant, available today).
2. **Last time** card: last visit's items with **Reorder** → items are re-validated; unavailable ones are listed ("Mocha isn't available today"), the rest go to the cart.
3. Place order without identity step.

### F-G3 — Returning guest on a new device

1. At first order, enters phone → OTP → verified → system finds the existing customer record at this tenant → "Welcome back, Riya" → history becomes available on this device.
2. If the entered name differs from the stored name, keep the stored name and offer "Not Riya? Update name" (to avoid accidental overwrite by someone else at the table).

### F-G4 — Joining a table that already has an open session

1. Second guest at the same table scans the same QR.
2. Menu loads; a small banner says "Your table has 1 order in progress · View".
3. Their orders join the same session; the session view shows all orders. Each order shows the name of the guest who placed it (first name only).

### F-G5 — Service request

1. Header menu or session screen → "Call waiter" / "Water" / "Request bill" etc.
2. Request sent; button shows "Sent · we've let them know"; repeated taps within 60 s show the same state (GST-SRV-003).
3. When staff acknowledge, guest sees "On the way".

### F-G6 — Cancelling

1. G-07 → "Cancel order" visible only while order status is `placed` or `confirmed`.
2. Confirm dialog → `POST /orders/{id}/cancel` → status "Cancelled".
3. If the kitchen started preparing in the meantime, the server rejects with `ORDER_NOT_CANCELLABLE`; the guest sees "The kitchen has already started on this. Ask your waiter if you need to change it" + "Call waiter".

### F-G7 — Offline at the table

1. Connection drops while browsing: ConnectionBanner "You're offline — your cart is saved". Menu, search, item sheets, and cart keep working (menu snapshot and images already loaded are cached).
2. "Place order" while offline: order saved to outbox; status shows "Waiting for connection… we'll send it automatically". The guest can't accidentally send twice.
3. Connection returns: outbox sends with the same idempotency key → "Sent" → normal flow.
4. If the outbox can't send for 2 minutes: show "Still can't connect. You can also ask your waiter" with a "Call waiter" button (which itself queues) and keep retrying.

### F-G8 — Room service (hotel)

Same as F-G1 with: spot label "Room 304", "Room service" menu, optional "Charge to room" payment option at the bill step (P1), delivery status "On the way to your room" instead of "Served".

---

## 2. Core staff/admin flows

### F-A1 — Owner onboarding (setup checklist)

1. Open invite link → set password → accept terms.
2. **Business details**: display name, legal name, GSTIN, FSSAI, address, country, currency.
3. **Branding**: logo, cover, theme preset, accent, fonts → live preview.
4. **First outlet**: name, address, time zone, opening hours, business-day cutoff.
5. **Taxes**: choose a preset ("Restaurant 5 % GST: CGST 2.5 % + SGST 2.5 %") or custom; price display inclusive/exclusive; service charge; rounding.
6. **Menu**: create categories and items (or import spreadsheet), upload photos.
7. **Spots & QR**: bulk-create tables/rooms → choose card template → download PDF.
8. **Staff**: invite manager/cashier/waiter/kitchen; pair KDS device.
9. **Test order**: place a test order from a phone (test mode) to see it on KDS.
10. **Go live**: turns off test mode and deletes test orders after confirmation.

### F-A2 — Daily service (manager / waiter)

1. Open app → **Floor view** (A-10) shows all spots with states.
2. New order arrives → sound → **Orders board** (A-11) card in "New" (if manual confirmation) → Confirm → kitchen gets the ticket.
3. Kitchen bumps "Ready" on KDS → board card moves to Ready → waiter serves → taps "Served".
4. Service requests appear in the Requests panel; waiter taps "On it" (acknowledge) then "Done".
5. Bill requested → spot turns blue on floor view → cashier opens **Bill** (A-13) → applies discount if needed → records payment → **Settle & clear**.
6. End of day → **Day close** (A-15) → review summary, count cash, close day.

### F-A3 — Menu change: add a dish with a photo (owner on phone)

1. Menu studio → category → "+ Add item".
2. Name, price, description, dietary type → "Add photo" → camera → crop 4:3 → upload starts in background (progress ring on thumbnail).
3. Optional: variants, modifiers (pick existing group "Milk options"), tags, prep time, station.
4. "Save" (draft) → "Publish" (or auto-publish if enabled) → toast "Live on the menu".
5. Guests see the new dish within 60 s (snapshot pointer TTL).

### F-A4 — Mark items sold out during service

1. Availability screen (A-22) → search "paneer" → toggle "Paneer Tikka" off → choose "Until end of day".
2. Change is live instantly; guests with it in cart see a notice at validation; open item sheets show "Just sold out".

### F-A5 — Price change in bulk

1. Menu studio → select category "Beverages" → "Change prices" → "+8 %", round to nearest ₹5 → preview table (old → new) → Apply → Publish.
2. Audit log records each price change.

### F-A6 — Move / merge tables

1. Floor view → spot → "Move to…" → pick free spot → session moves; guest QR at the old spot now starts a fresh session; guests who are viewing the moved session see "Your table moved to Table 7" and keep following it.
2. "Merge with…" → pick another occupied spot → bills combine into one session; both spots point to the merged session until settled.

### F-A7 — Void a settled bill

1. Bills history → bill → "Void" (owner/manager) → reason (required) → confirm → reversal created; reports adjust; audit log entry.

### F-A8 — Platform admin creates a tenant

1. Platform → Tenants → "New tenant" → business name, owner name, phone/email, plan, trial end → Create.
2. System creates tenant, default roles, default settings, sends owner invite.
3. Platform admin can open setup in **support mode** (with owner permission) to help configure.

---

## 3. Guest screens (G)

Every screen has these states unless noted: **loading** (skeleton), **ready**, **offline** (banner + cached content), **error** (friendly message + action).

| ID | Screen | Contents | Special states |
|----|--------|----------|----------------|
| G-01 | Loading shell | Branded skeleton (cover, tabs, 4 cards) using theme colours | Slow > 3 s: subtle "Still loading… weak connection" line. Fails: G-E1 |
| G-02 | Menu | CoverHeader, welcome / "Your usual" row, CategoryTabs, search & filter row, sections of ItemCards, CartBar | Closed (ordering off, next opening time), Ordering paused, Empty menu ("Menu coming soon"), Search no results ("Nothing for 'xyz' · Clear search") |
| G-03 | Item sheet | Hero photo, name, marks, badges, description, allergens, variants, modifier groups, instructions, stepper, Add button | Sold out (button disabled "Sold out"), Option sold out (option disabled), Unmet required group (highlight) |
| G-04 | Cart | Lines, order note, upsell row (P1), bill preview, Place order | Empty cart ("Your cart is empty · Browse menu"), Validation changes (inline change list + "Update cart"), Offline (Place order queues), Ordering closed |
| G-05 | Your details | Name, phone, consent text, marketing checkbox, Continue | Validation errors inline; "Skip" if anonymous allowed |
| G-06 | OTP | 6-digit input, number shown with "Change", resend countdown | Wrong code (shake + "That code isn't right · 3 tries left"), Expired, Too many attempts (wait time), SMS delayed ("Didn't get it? Resend in 0:25") |
| G-07 | Order status | Order no, timeline, items, ETA, actions (Add more, Your table, Cancel if allowed, Call waiter) | Waiting for confirmation, Declined (reason + options), Sending/queued (offline), Cancelled |
| G-08 | Your table (session & bill) | Orders list with statuses, bill summary, Request bill / Pay at counter, (Pay online in Phase 8) | Bill requested, Partially paid, Session moved/merged notice, Session closed ("This table's bill was settled. Scan again to start a new order") |
| G-09 | Paid & receipt | Thank you message, receipt (bill number, items, taxes, payments), Share/Download | — |
| G-10 | Feedback (P1) | 1–5 rating, optional comment, item thumbs | Submitted ("Thank you") |
| G-11 | Profile | Name, phone (masked), My orders, Privacy & data, Log out, Delete my data | Not identified: "Add your number to see your orders" |
| G-12 | My orders | Past visits list; visit detail with Reorder | Empty |
| G-13 | About | Business info, hours, address (map link), contact, Wi-Fi, social links | — |
| G-14 | Service requests | Request buttons grid | Request sent / acknowledged |
| G-E1 | Can't load | "We can't load the menu right now" + Retry; if a cached snapshot exists, show it instead with an offline banner | — |
| G-E2 | QR inactive | "This code is no longer active. Please ask the staff for help." (branded if the tenant is known) | — |
| G-E3 | Business unavailable | Tenant suspended/closed permanently | — |

---

## 4. Admin screens (A)

| ID | Screen | Main contents | Key states |
|----|--------|---------------|------------|
| A-01 | Login | Email/phone + password; phone OTP login; forgot password | Locked out (rate limit), 2FA step (owners, P1) |
| A-02 | Setup checklist | Steps with progress, preview phone | Completed |
| A-03 | Dashboard | Period/outlet selectors, Insight cards, Today tiles, charts, top/bottom items, peak heatmap | No data yet ("Insights appear after your first week"), Partial data |
| A-04 | Reports | Report list, filters, table + chart, export | Large export → background job with notification |
| A-10 | Floor view | Areas tabs, spot tiles by state, legend, quick actions | Empty (no spots → link to create) |
| A-11 | Orders board | Kanban columns, filters, sound toggle, connection indicator | Reconnecting banner, Ordering paused banner |
| A-12 | Order detail | Items, modifiers, notes, guest, timeline, actions | Cancelled/declined with reason |
| A-13 | Session & bill | Orders, lines, discounts, taxes, payments, actions: discount, remove service charge, record payment, settle & clear, print, move/merge | Partially paid, Settled, Voided |
| A-14 | Record payment | Method, amount (prefilled due), reference, cash tendered → change | Overpayment warning, split |
| A-15 | Day close | Summary, payment method totals, cash count input, open sessions list | Open sessions block close (with override for owner) |
| A-16 | Waiter order entry | Spot picker, fast menu with search and recents, cart, place order | — |
| A-17 | Service requests | List with age, acknowledge/done | — |
| A-20 | Menu studio — list | Menus, categories (drag reorder), items with inline price, availability, photo thumb, status (draft/published) | Unpublished changes bar ("5 changes · Review & publish") |
| A-21 | Item editor | Tabs: Details, Photos, Variants & options, Availability & schedule, Advanced (station, tax, prep time, SKU) | Validation errors, upload in progress |
| A-22 | Availability | Search + big toggles for items/variants/options; "sold out until" chooser | — |
| A-23 | Modifier groups | List, editor, "used by" items | — |
| A-24 | Menus & schedules | Menu list, assignment to outlets/areas/spot types, time ranges | Overlap warning |
| A-25 | Publish review | Diff list (added, changed, removed), Publish | Validation blocks |
| A-26 | Version history | Versions, restore | — |
| A-30 | Branding | Logo, cover, theme, accent, fonts, welcome text, preview | Contrast adjusted notice |
| A-31 | Outlets | List, editor (hours, closures, cutoff) | — |
| A-32 | Areas & spots | Areas, spots table, bulk create, enable/disable, regenerate QR | — |
| A-33 | QR designer | Templates, preview, download (selected/all) | Generating PDF progress |
| A-40 | Customers | Table with filters and search | Empty |
| A-41 | Customer profile | Info, stats, visit timeline, notes, consent, delete data | Deleted (anonymised) |
| A-50 | Staff | List, invite, roles, outlets, deactivate | Pending invite |
| A-51 | Roles & permissions | Role list, permission matrix editor (custom roles P1) | — |
| A-52 | Devices | KDS / counter devices paired, revoke | — |
| A-53 | Audit log | Filterable event list with before/after | — |
| A-60 | Settings | Business profile, taxes, service charge, rounding, ordering rules, service request types, notifications, legal, languages | — |
| A-61 | Plan & usage | Current plan, limits usage, contact for upgrade | Near-limit warnings |
| A-70 | Quick switch | PIN pad to switch staff user on shared devices | Wrong PIN, locked |

## 5. Kitchen screens (K)

| ID | Screen | Contents |
|----|--------|----------|
| K-01 | Pair device | Shows a 6-digit pairing code; manager enters it in A-52 (or scans), device becomes paired to outlet + station |
| K-02 | Tickets | Ticket grid, station filter, connection light, all-day counts toggle, recall |
| K-03 | Settings (local) | Sound choice/volume, columns, font size, theme |

## 6. Platform screens (P)

| ID | Screen |
|----|--------|
| P-01 | Tenants list |
| P-02 | Tenant detail (plan, flags, limits, outlets, usage, status actions, support access) |
| P-03 | Plans & features |
| P-04 | System health |
| P-05 | Platform audit log |
| P-06 | Announcements (P1) |

---

## 7. Navigation maps

### Guest
```
Menu (home)
 ├─ Item sheet
 ├─ Search / filters (inline)
 ├─ Cart ─▶ Details ─▶ OTP ─▶ Order status
 ├─ Header menu
 │   ├─ Your table (session & bill) ─▶ Paid & receipt ─▶ Feedback
 │   ├─ My orders ─▶ Visit detail
 │   ├─ Service requests
 │   ├─ Profile
 │   └─ About
```
Deep links: `/q/<token>` (entry), `/o/<orderId>` (order status, requires device/session ownership), `/r/<receiptToken>` (receipt share link, read-only).

### Admin (by role, default)
- **Owner/Manager**: Dashboard · Floor · Orders · Menu · Customers · Reports · Staff · Settings
- **Cashier**: Floor · Bills · Orders · Day close
- **Waiter**: Floor · Orders · Requests · New order
- **Kitchen**: KDS only

---

## 8. Screen-level performance notes

- G-02 must render from the server HTML before any JavaScript runs (ItemCards are plain HTML + CSS). JavaScript hydrates interactivity afterwards.
- G-03 item sheet code is part of the initial bundle (it's the first interaction); G-05/G-06 identity code and the realtime client are lazy-loaded when the cart opens.
- Admin charts and the image cropper are lazy-loaded per route.
