# 02 — Product Requirements Document (PRD)

Version 1.0 · Owner: Founder · Status: Draft for review

This document lists **what** the product must do. **How** it is built lives in docs 05–09. Each requirement has a stable ID, a priority, and acceptance criteria.

**Priority**
- **P0** — required for the first sellable release.
- **P1** — required soon after launch; designed for now so it fits without rework.
- **P2** — later; listed so the data model and UI leave room for it.

**Phase** refers to [15-development-roadmap.md](./15-development-roadmap.md).

---

## 1. Goals and non-goals

### Goals
1. A guest can go from scanning a QR code to a placed order in under a minute, on a weak connection, without help.
2. A business owner can run the entire product (menu, prices, photos, tables, QR codes, staff, taxes, branding) without contacting us.
3. The kitchen receives every order exactly once, within seconds.
4. The business knows its customers and what they ordered before, with the customer's consent.
5. The dashboard explains the business in plain sentences.
6. The product looks and feels premium on every screen.

### Non-goals (this version)
- Building our own payment gateway (D-03).
- Ingredient-level inventory, purchasing, supplier management.
- Delivery aggregator integration, table reservations, loyalty wallets.
- Native mobile apps.

---

## 2. Personas and key scenarios

### Persona A — Riya, 27, guest at a busy cafe
On a crowded Saturday with weak Wi-Fi. Wants to see photos, order a coffee with oat milk, add a croissant later, and pay without waiting.

**Scenario**: scans QR → menu appears instantly with the cafe's branding → taps "Iced Latte" → chooses Large + Oat milk → adds to cart → later adds a croissant → places order → enters name + phone → OTP auto-fills → sees "Preparing" → later taps "Add more" for a second round → at the end taps "Request bill" → pays at counter.

### Persona B — Mr. Sharma, 58, guest at a family restaurant
Not very comfortable with apps. Needs large text, clear buttons, and the option to just call a waiter.

**Scenario**: scans QR → sees big photos and clear prices → taps "Call waiter" → waiter arrives and places the order for the table from their phone → Mr. Sharma watches the order appear on his own screen.

### Persona C — Arjun, 34, owner of two cafes
Changes the menu often, wants to know what sells, hates calling vendors.

**Scenario**: uploads a new seasonal drink with photo from his phone, sets price ₹280, sets it to show only 4–10 PM, prints a QR for a new table, checks the dashboard and reads "Weekend revenue up 22%; Iced Latte is your top seller; Mocha has not sold in 10 days".

### Persona D — Meena, 41, manager of a 60-room hotel
Wants room service via QR in every room plus the rooftop restaurant.

**Scenario**: creates two areas (Rooms, Rooftop), bulk-creates 60 room spots named "Room 101"…, downloads one PDF of all QR cards, assigns room-service orders to the kitchen, marks dishes sold out when stock runs out.

### Persona E — Kitchen team at a busy restaurant
Need a screen that is readable from two metres away, loud for new orders, and impossible to miss.

---

## 3. Guest experience requirements

### 3.1 QR entry and spot recognition (`GST-QR`)

| ID | Pri | Requirement |
|----|-----|-------------|
| GST-QR-001 | P0 | Scanning a spot's QR opens the guest web app for that exact outlet and spot in the phone's default browser. No app install, no login. |
| GST-QR-002 | P0 | The QR contains a short link with an unguessable token (≥ 10 random characters, base-62). The token reveals nothing about the tenant, outlet, or spot number. |
| GST-QR-003 | P0 | The first screen shows the business logo, name, and spot label ("Table 12", "Room 304") so the guest knows the order goes to the right place. |
| GST-QR-004 | P0 | If a QR is disabled, deleted, or the outlet is closed, the guest sees a friendly branded screen explaining why, with "View menu" still possible if the business allows browsing while closed. |
| GST-QR-005 | P0 | If the outlet is outside opening hours, the menu is viewable (configurable) but ordering is disabled, with the next opening time shown. |
| GST-QR-006 | P1 | An outlet-level QR (not tied to a spot, e.g. on a poster or counter) lets guests browse and, if enabled, order for pickup at counter ("Takeaway / counter pickup" mode). |
| GST-QR-007 | P0 | Re-scanning the same QR, or reopening the link later, resumes the guest's cart and active session if still open. |
| GST-QR-008 | P1 | "Add to home screen" is offered only after a successful order, never on first load. |

**Acceptance**: scanning any active QR on a mid-range Android on 4G shows the branded menu shell in < 1.5 s p75 (NFR-PERF-001). Scanning a deleted QR never shows another tenant's data.

### 3.2 Menu browsing (`GST-MENU`)

| ID | Pri | Requirement |
|----|-----|-------------|
| GST-MENU-001 | P0 | Show the menu currently available for the outlet at the current local time (menu schedules, ADM-MENU-010). If several menus overlap (e.g. "All Day" + "Breakfast"), merge them with section headers. |
| GST-MENU-002 | P0 | Categories appear as a sticky horizontal tab bar. Tapping a tab smooth-scrolls to it; scrolling updates the active tab (scroll-spy). |
| GST-MENU-003 | P0 | Each item card shows: photo (if any), name, short description (2 lines max, truncated), price (or "from ₹X" if variants), dietary mark (veg / non-veg / egg / vegan), and badges (Bestseller, Chef's pick, New, Spicy level). |
| GST-MENU-004 | P0 | Sold-out items stay visible, greyed, labelled "Sold out", and cannot be added. Hidden items never appear. |
| GST-MENU-005 | P0 | Search across item names, descriptions, and tags, with results as you type (client-side, works offline once menu is loaded). Tolerates small typos ("panner" finds "Paneer"). |
| GST-MENU-006 | P0 | Filters: Veg only, Non-veg, Egg, Vegan, plus tenant-defined tags (e.g. "Gluten free", "Jain"). Filters combine with search. Filter choice is remembered for the session. |
| GST-MENU-007 | P0 | Item layout options chosen by the tenant per category: **photo list** (default), **photo grid** (2 columns), **text list** (no photos, for drinks/bar). |
| GST-MENU-008 | P0 | Images load progressively: blurred placeholder first, then the right size for the screen (doc 08 §5). The page never jumps while images load. |
| GST-MENU-009 | P0 | Prices are shown as the business configured them (tax-inclusive or tax-exclusive, ADM-SET-004). If exclusive, a small note says "Prices exclude taxes". |
| GST-MENU-010 | P1 | "Recommended for you" row at the top for recognised returning guests (their past favourites) and "Popular here" for everyone else. |
| GST-MENU-011 | P0 | **English / हिंदी switch** in the header (D-11). Default is English. Switching is instant (no reload) and remembered on the device. All app text (buttons, labels, messages, errors) switches; dish names and descriptions switch where the business has added Hindi text and fall back to English otherwise. |
| GST-MENU-012 | P1 | Allergen information per item, displayed in item details with icons and text. |
| GST-MENU-013 | P0 | Menu changes made by the admin (price, sold out) appear for guests within 60 s without a reload; sold-out changes affecting a cart item appear immediately at checkout validation (GST-CART-008). |
| GST-MENU-014 | P1 | Upsell suggestions configured by the tenant ("Goes well with: Garlic bread") in the item sheet and the cart. |

### 3.3 Item details and customisation (`GST-ITEM`)

| ID | Pri | Requirement |
|----|-----|-------------|
| GST-ITEM-001 | P0 | Tapping an item opens a bottom sheet (mobile) with large photo, full description, dietary mark, allergens, variants, modifier groups, special instructions field, quantity stepper, and "Add · ₹total" button. |
| GST-ITEM-002 | P0 | Variants are a required single choice (e.g. Half / Full). The default variant is preselected. |
| GST-ITEM-003 | P0 | Modifier groups support: required/optional, min and max selections, single choice (radio) or multiple choice (checkbox), per-option price, per-option quantity (P1, e.g. "Extra shot ×2"), and per-option sold-out. |
| GST-ITEM-004 | P0 | The "Add" button shows the live total as options change and is disabled until required groups are satisfied; tapping it while disabled scrolls to and highlights the first unmet group. |
| GST-ITEM-005 | P0 | Special instructions: free text, max 140 characters, can be disabled per item by the tenant. |
| GST-ITEM-006 | P0 | Items without variants or modifiers can be added directly from the menu list with a "+" button, without opening the sheet. |
| GST-ITEM-007 | P0 | The sheet can be dismissed by swipe-down, tapping outside, or the back button (Android back must close the sheet, not leave the page). |
| GST-ITEM-008 | P1 | Combo / meal items: an item composed of choices from other items (e.g. "Burger meal: pick burger + side + drink"). Modelled as modifier groups whose options reference items. |

### 3.4 Cart (`GST-CART`)

| ID | Pri | Requirement |
|----|-----|-------------|
| GST-CART-001 | P0 | A floating cart bar appears after the first item is added: item count, total, "View cart". It animates in and updates with a subtle count bump on every change. |
| GST-CART-002 | P0 | Cart screen lists each line: item, variant, modifiers, instructions, quantity stepper, line total. Lines with identical configuration merge; different configurations are separate lines. |
| GST-CART-003 | P0 | Increase/decrease quantity with immediate visual feedback. Decreasing to zero removes the line with an "Undo" toast for 5 s. |
| GST-CART-004 | P0 | Tapping a line reopens the item sheet to edit its options. |
| GST-CART-005 | P0 | Cart is saved on the device (survives refresh, tab close, network loss, re-scan) for 12 hours or until ordered. Cart is tied to the outlet; scanning a different outlet's QR keeps carts separate. |
| GST-CART-006 | P0 | Cart shows a bill preview: item subtotal, discounts (if any auto-applied), taxes, service charge (optional and removable, D-15), rounding, and total. Calculated with the same rules as the server (doc 06 §8). |
| GST-CART-007 | P0 | Order notes for the whole order (e.g. "Serve starters first"), max 200 characters. |
| GST-CART-008 | P0 | Before placing an order, the cart is validated against the current menu. If an item became sold out, its price changed, or an option was removed, the guest sees exactly which lines changed and must confirm. Never silently change a price. |
| GST-CART-009 | P0 | Maximum quantity per line (default 20) and per order (default 50 items) to stop accidental or malicious huge orders; tenant can change both. |
| GST-CART-010 | P2 | **Shared table cart** (D-09): everyone at the spot sees one live cart with each person's name/avatar on their lines; anyone can place the order. |

### 3.5 Placing orders and the table session (`GST-ORD`)

| ID | Pri | Requirement |
|----|-----|-------------|
| GST-ORD-001 | P0 | "Place order" submits the cart as one order (a round) into the spot's active table session. If no session is open, one is created. |
| GST-ORD-002 | P0 | Placing an order is **idempotent**: a double tap, a retry after a timeout, or a resend after going offline creates exactly one order (doc 09 §5). |
| GST-ORD-003 | P0 | On a weak or lost connection, the order is queued on the device and sent automatically when the connection returns. The guest sees "Sending your order…" with honest status, never a false "Confirmed". |
| GST-ORD-004 | P0 | Guest identity (GST-ID) is requested at the first order of a session if the tenant requires it; returning recognised guests skip this step. |
| GST-ORD-005 | P0 | After placing, the guest sees an order status screen with a live timeline: **Sent → Confirmed → Preparing → Ready → Served**. Status updates arrive in realtime, with automatic fallback to periodic refresh. |
| GST-ORD-006 | P0 | If the tenant requires staff confirmation for the first order of a session (D-14), the status shows "Waiting for the restaurant to confirm" and the guest is notified when confirmed or declined (with reason). |
| GST-ORD-007 | P0 | "Add more items" returns to the menu; the next order joins the same session and bill. |
| GST-ORD-008 | P0 | **Session view** ("Your table"): all orders in the session, each with its status, and the running bill. Anyone who scans the same spot's QR while the session is open sees the session's orders and bill, so the whole table can follow along. |
| GST-ORD-009 | P0 | Guests can cancel an order (or a line) only while it is in "Sent" or "Confirmed" and not yet "Preparing". After that, they must ask staff (with a "Call waiter" shortcut). |
| GST-ORD-010 | P0 | Estimated preparation time shown per order if the tenant sets prep times per item (max of item prep times + kitchen load factor, ADM-ORD-012). |
| GST-ORD-011 | P1 | Browser notification / vibration / sound when an order is ready (only if the guest allows). |
| GST-ORD-012 | P0 | A session closes when staff settle the bill and clear the spot (ADM-BIL-006), or automatically after an idle timeout set by the tenant (default 4 hours since last activity). New scans after closing start a fresh session; old guests can no longer see the new session. |
| GST-ORD-013 | P1 | **Counter pickup mode** (outlet QR, GST-QR-006): order gets a short pickup number; guest is notified when ready. |
| GST-ORD-014 | P1 | Room service mode for hotel rooms: shows "Delivered to Room 304", option "Charge to room" (recorded as a payment method for the hotel to settle at checkout; no PMS integration in v1). |

### 3.6 Guest identity, history, and recognition (`GST-ID`)

| ID | Pri | Requirement |
|----|-----|-------------|
| GST-ID-001 | P0 | Guest enters only their **name** and **mobile number** (D-17). The number is an Indian mobile number: `+91` is fixed, 10 digits starting with 6–9. No other personal fields are ever asked. |
| GST-ID-002 | P0 | Phone is verified with a 6-digit OTP sent by SMS (D-05). On Android, the OTP auto-fills using the Web OTP API; on iOS, the keyboard suggests it via `autocomplete="one-time-code"`. |
| GST-ID-003 | P0 | OTP rules: valid 5 minutes, max 5 attempts, resend allowed after 30 s, max 3 sends per number per 15 minutes, max 10 per device per hour (doc 10 §6). |
| GST-ID-004 | P0 | A clear consent line is shown before sending the OTP: what we store (name, phone number, orders — nothing else), why (order updates, history, the business recognising you), and a link to the privacy notice. Optional separate checkbox for marketing messages (unticked by default). |
| GST-ID-005 | P0 | After verification, the device is remembered for that tenant for 180 days (secure, httpOnly cookie). Next visit: "Welcome back, Riya" and no OTP needed. |
| GST-ID-006 | P0 | Recognised guests see **"Your usual"**: their most ordered items at this tenant, and **"Last time you had…"** with one-tap **Reorder** (re-validated against today's menu and prices). |
| GST-ID-007 | P0 | "My orders" screen: past visits at this tenant (date, outlet, items, total), each with Reorder. |
| GST-ID-008 | P0 | Tenants can switch OTP off (D-05). Then name + phone are stored as **unverified**, history is shown only on the same device that entered them, and the admin sees an "Unverified" label. |
| GST-ID-009 | P0 | Guest can log out on the device, and can request deletion of their data at this tenant from the profile screen (doc 10 §8). |
| GST-ID-010 | P1 | WhatsApp as an OTP channel where available (fallback to SMS). |
| GST-ID-012 | P0 | Guests can skip identity entirely if the tenant allows anonymous ordering (setting); they can still order and pay at counter. |

### 3.7 Bill and payment (`GST-PAY`)

| ID | Pri | Requirement |
|----|-----|-------------|
| GST-PAY-001 | P0 | "View bill" shows the full session bill: each order's items, subtotal, discounts, taxes (by tax type, e.g. CGST 2.5 %, SGST 2.5 %), service charge (optional, removable), rounding, total, amount paid, amount due. |
| GST-PAY-002 | P0 | **Pay at counter**: guest taps "Pay at counter"; staff are notified ("Table 12 wants to pay") and the bill is marked "Bill requested". |
| GST-PAY-003 | P1 (Phase 8) | **Pay online** through the payment plug-in (doc 12): full amount or (P2) split equally / by items. |
| GST-PAY-004 | P0 | After settlement, the guest sees "Paid — thank you" and a digital receipt (shareable link, downloadable PDF in P1). |
| GST-PAY-005 | P1 | Tip option (preset percentages + custom) if enabled by the tenant. Tips are recorded separately from revenue. |
| GST-PAY-006 | P0 | The bill is always calculated by the server. The guest app only displays it. |

### 3.8 Service requests (`GST-SRV`)

| ID | Pri | Requirement |
|----|-----|-------------|
| GST-SRV-001 | P0 | Buttons: **Call waiter**, **Request bill**, plus tenant-defined requests (e.g. "Water", "Cutlery", "Clean table", "Extra towels" for rooms). |
| GST-SRV-002 | P0 | Requests appear instantly on the staff app/admin orders board with spot and time; staff mark them done; guest sees "On the way". |
| GST-SRV-003 | P0 | Rate limit: the same request type from the same spot can't be sent again for 60 s ("We've let them know"). |

### 3.9 Feedback (`GST-FBK`)

| ID | Pri | Requirement |
|----|-----|-------------|
| GST-FBK-001 | P1 | After settlement, a one-tap rating (1–5) with optional comment and optional per-item thumbs up/down. |
| GST-FBK-002 | P1 | Low ratings (≤ 2) notify the manager immediately. |
| GST-FBK-003 | P2 | High ratings can show a "Review us on Google" link set by the tenant. |

---

## 4. Business admin requirements

The admin app is used on laptops, tablets, and phones. Every screen must work on a phone (owners manage from anywhere), but dense screens (menu studio, reports) are optimised for larger screens.

### 4.1 Onboarding and setup (`ADM-ONB`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-ONB-001 | P0 | Owner receives an invite (email/SMS link) created by the platform admin, sets a password, and lands in a **setup checklist**: business details → branding → first outlet → taxes → menu → spots & QR → staff → go live. |
| ADM-ONB-002 | P0 | Each checklist step shows progress and can be skipped and returned to. A live phone preview shows the guest menu as the owner edits. |
| ADM-ONB-003 | P1 | Menu import from a spreadsheet (CSV/XLSX template) with validation report (row, problem, fix). |
| ADM-ONB-004 | P2 | Menu import from a photo/PDF of an existing menu (AI-assisted extraction, reviewed by owner before saving). |
| ADM-ONB-005 | P0 | "Test mode" for an outlet: orders are marked as tests, excluded from reports, and can be deleted in one click before going live. |

### 4.2 Branding (`ADM-BRD`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-BRD-001 | P0 | Upload logo (square and wide versions), cover image, and choose brand colour (accent). |
| ADM-BRD-002 | P0 | Choose a theme preset (doc 03 §9): **Ivory** (light, warm), **Noir** (dark, elegant), **Garden** (light, green-tinted), **Coastal** (light, blue-tinted), or custom accent. |
| ADM-BRD-003 | P0 | Choose a font pairing from curated options (doc 03 §4.3). No arbitrary font upload (protects performance and quality). |
| ADM-BRD-004 | P0 | The system checks colour contrast automatically and adjusts the accent's text colour (or warns) so the guest app always meets WCAG AA. |
| ADM-BRD-005 | P0 | Welcome message, "About us" text, address, opening hours, contact phone, social links, Wi-Fi name/password (optional, shown to guests). |
| ADM-BRD-006 | P0 | Live preview of the guest app in a phone frame, updating as settings change. |

### 4.3 Outlets, areas, spots, and QR codes (`ADM-TBL`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-TBL-001 | P0 | Create outlets with name, address, time zone, opening hours (per weekday, multiple ranges, special closures/holidays), business-day cutoff (default 04:00), currency. |
| ADM-TBL-002 | P0 | Create areas within an outlet ("Ground floor", "Rooftop", "Rooms — Floor 3"). |
| ADM-TBL-003 | P0 | Create spots one by one or in bulk ("Table 1–20", "Room 101–130", with prefix, start, end, and step). Each spot has a type (table / room / counter / other), label, area, capacity (optional). |
| ADM-TBL-004 | P0 | Each spot gets a QR code automatically. Admin can **regenerate** a spot's QR (old code stops working immediately and shows "This code is no longer active") and **disable/enable** a spot. |
| ADM-TBL-005 | P0 | **QR card designer**: choose from print templates (table tent A6, sticker 8 × 8 cm, A4 poster, room card), with logo, spot label, short instruction ("Scan to order"), optional Wi-Fi details; preview, then download as print-ready PDF (one or all spots, crop marks optional) or PNG/SVG. |
| ADM-TBL-006 | P0 | Live **floor view**: grid of spots in each area showing state — free, ordering, waiting for food, bill requested, needs attention — with session total and time seated. Tapping a spot opens its session. |
| ADM-TBL-007 | P1 | Drag-and-drop floor layout (arrange spots to match the real room). |
| ADM-TBL-008 | P0 | Staff can **move** a session to another spot (guests changed tables) and **merge** two sessions (tables joined). |
| ADM-TBL-009 | P0 | QR scan analytics per spot: scans, orders, conversion. |

### 4.4 Menu studio (`ADM-MENU`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-MENU-001 | P0 | Create, edit, reorder (drag and drop), hide, and delete **categories**. Category fields: name, description, image (optional), layout (GST-MENU-007), visibility. |
| ADM-MENU-002 | P0 | Create and edit **items**: name, description (rich text limited to bold/italic/line breaks), photos (up to 5, first is cover), dietary type, tags, badges, spice level (0–3), allergens, preparation time, special-instructions allowed, max quantity, display order, kitchen station (ADM-ORD-011), tax group. |
| ADM-MENU-003 | P0 | **Variants** per item (e.g. Half ₹180 / Full ₹320), each with its own price and SKU code (optional). An item with no variants has one implicit default variant. |
| ADM-MENU-004 | P0 | **Modifier groups** reusable across items (e.g. "Milk options" used by 12 coffees): name, required/optional, min/max, single/multiple, options with prices, option sold-out, option default. Editing a shared group shows "Used by 12 items". |
| ADM-MENU-005 | P0 | **Photo upload** from phone camera or computer: drag-and-drop, crop to the required ratio (4:3 card, 1:1 thumbnail) with a cropping tool, automatic compression and resizing in the background, progress indicator, retry on failure. Accepted: JPEG, PNG, WebP, HEIC (converted). Max 20 MB per original. |
| ADM-MENU-006 | P0 | **Sold out** toggle per item, per variant, per modifier option — available as a one-tap switch in a dedicated "Availability" screen optimised for phones (search + big toggles), with "Sold out until end of day" (auto-restores at business-day cutoff) or "until I turn it back on". |
| ADM-MENU-007 | P0 | **Price management**: edit price inline in the item list; bulk price change (select items → +/- amount or %, with rounding to nearest ₹1/₹5/₹10) with preview before applying. |
| ADM-MENU-008 | P0 | **Outlet overrides**: a tenant with several outlets shares one menu library; each outlet can override price, availability, and visibility per item/variant. |
| ADM-MENU-009 | P0 | **Menus** (e.g. "All day", "Breakfast", "Bar", "Room service") are collections of categories, assigned to outlets and to spot types or areas (e.g. "Room service" menu only for room spots). |
| ADM-MENU-010 | P0 | **Menu schedules**: each menu has active days and time ranges (e.g. Breakfast Mon–Sun 07:00–11:00). Items can also have their own time windows ("Happy hour pricing" in P2). |
| ADM-MENU-011 | P0 | **Draft and publish**: edits are saved as a draft; "Publish" makes them live for guests (creates a new menu snapshot version). Small operational changes (sold out, price change, visibility) can be set to publish instantly (setting, default on). Show "3 unpublished changes" with a diff list. |
| ADM-MENU-012 | P0 | **Version history**: list of published versions with who/when; restore a previous version as a new draft. |
| ADM-MENU-013 | P0 | Duplicate item / category / menu. |
| ADM-MENU-014 | P0 | Validation: an item cannot be published without a name and price; a required modifier group must have at least one available option; warnings (not blocks) for missing photos or descriptions. |
| ADM-MENU-015 | P0 | **Hindi text** for categories, items, variants, modifier groups and options: a "Hindi" field next to each English field, a "missing Hindi" filter in the item list, and preview of the guest app in Hindi. Optional — guests see English where Hindi is empty. |
| ADM-MENU-016 | P1 | Upsell links between items (GST-MENU-014). |
| ADM-MENU-017 | P1 | Export menu to CSV/XLSX; import updates (ADM-ONB-003). |
| ADM-MENU-018 | P2 | AI helpers: write a description from the dish name, suggest tags, enhance photo lighting. Always reviewed by the user before saving. |

### 4.5 Live orders and operations (`ADM-ORD`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-ORD-001 | P0 | **Orders board** (realtime): columns **New → Confirmed → Preparing → Ready → Served**, each card shows spot, order number, time since placed (turns amber after a threshold, red after a second threshold), items with modifiers and notes, guest name, and "First visit / Returning (5th visit)" badge. |
| ADM-ORD-002 | P0 | New orders play a sound (selectable, volume setting), show a notification badge, and flash the tab title. Sound keeps repeating every 30 s until a New order is acknowledged (configurable). |
| ADM-ORD-003 | P0 | Actions per order: confirm, decline (with reason: item unavailable, kitchen closed, suspicious order, other), mark preparing / ready / served, cancel item lines (with reason, permission-checked), print (P1). |
| ADM-ORD-004 | P0 | **Order acceptance mode** per outlet: auto-confirm all, manual confirm all, or **manual confirm first order of each session** (default, D-14). |
| ADM-ORD-005 | P0 | **Waiter ordering**: staff can create an order for any spot from the staff app with the same menu UI (fast search, recent items), optionally attaching a guest by phone. |
| ADM-ORD-006 | P0 | **Service requests** panel: open requests with spot, type, age; mark done. |
| ADM-ORD-007 | P0 | Filter board by area, station, status; search by spot or order number. |
| ADM-ORD-008 | P0 | Order history list with filters (date range, status, spot, staff, payment method) and order details including a full status timeline (who did what, when). |
| ADM-ORD-009 | P0 | Pause ordering for the whole outlet ("Kitchen busy — ordering paused for 20 min") or for a station; guests see a friendly message and can still browse. |
| ADM-ORD-010 | P0 | Boards and KDS keep working through brief disconnects; on reconnect they fetch everything missed and show a "Reconnected — 2 new orders" banner (doc 08 §7). |
| ADM-ORD-011 | P1 | **Kitchen stations** (e.g. Kitchen, Bar, Dessert): items are routed to stations; each KDS shows only its station's lines. |
| ADM-ORD-012 | P1 | Prep time tracking: actual time from Confirmed to Ready per item, used for estimates (GST-ORD-010) and insights. |

### 4.6 Kitchen Display System (`KDS`)

| ID | Pri | Requirement |
|----|-----|-------------|
| KDS-001 | P0 | Full-screen tablet view (landscape and portrait), large type readable at 2 m, high-contrast dark theme by default. |
| KDS-002 | P0 | Tickets in arrival order: spot, order number, elapsed time (colour-coded), items with quantities, modifiers, and notes highlighted. |
| KDS-003 | P0 | Tap item to strike it through (prepared); tap "Ready" (or "Bump") to move the whole ticket to Ready. "Recall" restores the last bumped ticket. |
| KDS-004 | P0 | Loud new-ticket sound; screen keeps awake (Wake Lock API); full-screen mode; works for a full shift without reloading. |
| KDS-005 | P0 | Connection indicator always visible: green (live), amber (reconnecting), red (offline > 10 s, with a loud alert). |
| KDS-006 | P0 | Logged in with a **device PIN pairing** (doc 10 §4), not a personal account; can be revoked from admin. |
| KDS-007 | P1 | All-day count view ("Paneer Tikka × 7 pending") for batch cooking. |
| KDS-008 | P1 | Thermal printer KOT printing via print bridge (D-10). |

### 4.7 Billing and settlement (`ADM-BIL`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-BIL-001 | P0 | Session bill view for any spot: all orders, lines, discounts, taxes, service charge, rounding, total, payments, due. |
| ADM-BIL-002 | P0 | Apply discount to the bill or to a line: percentage or fixed amount, with reason; limited by role (e.g. cashier max 10 %, manager max 30 %, owner unlimited; configurable). |
| ADM-BIL-003 | P0 | Remove service charge at guest request (always allowed, D-15). |
| ADM-BIL-004 | P0 | **Record counter payment**: method (cash, card on terminal, UPI on counter QR, room charge, other), amount, reference (optional). Supports **partial and split payments** (multiple payments per bill until due is zero). Cash payments compute change. |
| ADM-BIL-005 | P0 | Generate a bill number on settlement: per outlet, sequential per financial year (GST requirement), prefix configurable (e.g. `BND/25-26/000123`). |
| ADM-BIL-006 | P0 | **Settle & clear**: when due is zero, staff settle the bill and the spot returns to free; the session closes (GST-ORD-012). |
| ADM-BIL-007 | P0 | **Void** a settled bill (owner/manager only, with reason); creates a reversal record, never deletes. Bill numbers are never reused. |
| ADM-BIL-008 | P0 | Print bill (browser print with a receipt-sized layout, 58 mm and 80 mm) and share digital bill link by SMS/WhatsApp (P1). |
| ADM-BIL-009 | P1 | Coupons / offers: code-based or automatic (e.g. "10 % off above ₹1000", "Happy hour 4–7 PM 20 % off drinks"), validity dates, usage limits, per-customer limits. |
| ADM-BIL-010 | P0 | **End-of-day (day close) report** per outlet: sales by payment method, discounts, voids, taxes collected, open sessions warning; cashier can enter counted cash to show differences. |

### 4.8 Staff and roles (`ADM-STF`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-STF-001 | P0 | Invite staff by phone or email; staff set their own password (or login by phone OTP). |
| ADM-STF-002 | P0 | Built-in roles: **Owner, Manager, Cashier, Waiter, Kitchen** with default permissions (doc 10 §3). |
| ADM-STF-003 | P1 | Custom roles: copy a built-in role and toggle permissions. |
| ADM-STF-004 | P0 | Assign staff to one or more outlets (all outlets for owners). |
| ADM-STF-005 | P0 | Deactivate staff instantly (all their sessions end within 60 s). |
| ADM-STF-006 | P0 | Shared-device **quick switch**: on a counter tablet, staff switch user with a 4–6 digit personal PIN, so actions are attributed to the right person. |
| ADM-STF-007 | P0 | **Audit log**: every sensitive action (price change, discount, void, refund, staff change, settings change, data export, customer data view) with who, when, device, before/after values. Owners can view and filter it. |

### 4.9 Customers (`ADM-CUS`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-CUS-001 | P0 | Customer list: name, phone (masked by default for non-owners, e.g. `98•••••210`), verified badge, first visit, last visit, visits, total spend, average bill, favourite items. Search by name/phone; sort and filter (e.g. "not visited in 30 days", "spend > ₹5,000"). |
| ADM-CUS-002 | P0 | Customer profile: visit timeline with every order, favourite items, feedback given, notes added by staff (e.g. "Prefers window seat"), marketing consent status. |
| ADM-CUS-003 | P0 | On the orders board and session view, the guest's badge (returning, visit count, a short note) is visible to staff so they can greet them personally. |
| ADM-CUS-004 | P0 | Export customers (CSV) — owner only, audit-logged, includes only customers with appropriate consent for marketing exports. |
| ADM-CUS-005 | P0 | Delete a customer's personal data on request (anonymises name/phone, keeps order totals for accounting). |
| ADM-CUS-006 | P1 | Segments (saved filters) such as "Regulars", "At risk", "Big spenders", used by insights and future campaigns. |
| ADM-CUS-007 | P2 | Campaigns (SMS/WhatsApp) to consented segments. |

### 4.10 Dashboard and reports (`ADM-DSH`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-DSH-001 | P0 | **Today at a glance** (live): revenue so far, orders, average bill, guests served, open tables, compared with the same weekday last week, with a small sparkline by hour. |
| ADM-DSH-002 | P0 | **Insight cards** at the top: 3–6 plain-language insights ranked by importance, each with an explanation and a suggested action (doc 11). Example: "Cold Coffee hasn't sold in 12 days. Consider removing it or adding a photo." |
| ADM-DSH-003 | P0 | Period selector: today, yesterday, last 7 days, last 30 days, this month, custom; outlet selector (all / one). |
| ADM-DSH-004 | P0 | Metrics: revenue (gross, discounts, net, tax), orders, average order value, items sold, top and bottom items, category mix, peak hours heatmap (weekday × hour), payment method split, new vs returning customers, average prep time, average table turnover time, cancellation rate. |
| ADM-DSH-005 | P0 | Every number can be tapped to see the underlying list (e.g. revenue → bills). |
| ADM-DSH-006 | P0 | Reports: sales summary, item sales, category sales, tax report (GST by rate), discounts & voids, payments, staff performance (orders handled, discounts given), hourly sales. Export CSV/XLSX/PDF. |
| ADM-DSH-007 | P1 | Daily summary sent to owner by email/WhatsApp at day close. |
| ADM-DSH-008 | P2 | "Ask your data" — natural language questions answered from report data (D-12). |

### 4.11 Settings (`ADM-SET`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-SET-001 | P0 | Business profile: legal name, display name, GSTIN, FSSAI licence number (shown on bills), address, contact. |
| ADM-SET-002 | P0 | **Tax groups**: define taxes (name, rate, e.g. CGST 2.5 % + SGST 2.5 %), group them, assign groups to items/categories; default group per outlet. |
| ADM-SET-003 | P0 | Service charge: on/off, percentage, applies to dine-in only / all, label text. |
| ADM-SET-004 | P0 | Price display: tax-inclusive or tax-exclusive. |
| ADM-SET-005 | P0 | Rounding: none, nearest ₹1, nearest ₹0.50 (shown as a separate "Round off" line). |
| ADM-SET-006 | P0 | Ordering rules: acceptance mode (ADM-ORD-004), anonymous ordering allowed, OTP required, browse when closed, max items per order, session idle timeout, guest cancellation window. |
| ADM-SET-007 | P0 | Service request types (add/rename/reorder/disable). |
| ADM-SET-008 | P0 | Notification settings: sounds, who gets notified for bill requests, low ratings, day close summary. |
| ADM-SET-009 | P0 | Guest-facing legal: privacy notice text (default template provided), terms; contact for data requests. |
| ADM-SET-010 | P0 | Show or hide the Hindi switch for guests (default shown). |
| ADM-SET-011 | P1 | Admin app language: English (default) or Hindi, per staff user. Especially useful for kitchen and waiter screens. |

### 4.12 Notifications to staff (`ADM-NTF`)

| ID | Pri | Requirement |
|----|-----|-------------|
| ADM-NTF-001 | P0 | In-app realtime notifications: new order, order waiting too long, service request, bill requested, payment received (P8), low rating (P1). |
| ADM-NTF-002 | P1 | Web push notifications to staff phones (opt-in per device) for the same events. |

---

## 5. Platform admin requirements (`PLT`)

| ID | Pri | Requirement |
|----|-----|-------------|
| PLT-001 | P0 | Create a tenant: business name, owner contact, plan, outlets allowed, trial end date; sends owner invite. |
| PLT-002 | P0 | Tenant list with status (trial, active, past due, suspended, cancelled), plan, outlets, last activity, orders last 30 days. |
| PLT-003 | P0 | **Plan builder** (D-13): create, edit, and retire plans; each plan has a name, price, billing period (monthly / yearly), included features (feature flags), and limits (outlets, spots, staff, KDS devices, menu items, photo storage). Assign a plan to a tenant; override any feature or limit per tenant (e.g. a custom deal). Changing a plan never needs a code change. |
| PLT-004 | P0 | Suspend / reactivate tenant (guest QR shows "temporarily unavailable"; admin read-only; data kept). |
| PLT-005 | P0 | **Support access**: platform admin can view a tenant's admin in read-only mode, or with edit access only after the owner grants time-limited permission; every support session is audit-logged and visible to the owner. |
| PLT-006 | P0 | System health: API latency, error rates, realtime connections, job queue depth, SMS delivery rate (links to monitoring dashboards). |
| PLT-007 | P1 | Subscription invoices (manual in v1: generate invoice PDF, mark paid). |
| PLT-008 | P1 | Announcements shown in tenant admin (maintenance windows, new features). |
| PLT-009 | P0 | Data export of a whole tenant (for offboarding) and tenant deletion with a 30-day grace period. |

---

## 6. Non-functional requirements (summary)

Details and measurement methods: docs 08, 09, 10.

| ID | Requirement |
|----|-------------|
| NFR-PERF-001 | Guest menu visible (Largest Contentful Paint) < 1.5 s p75 on 4G mid-range Android; < 3.5 s on slow 3G. |
| NFR-PERF-002 | Guest app initial JavaScript ≤ 130 KB gzip on the menu route; total transfer for first view ≤ 400 KB excluding below-the-fold images. |
| NFR-PERF-003 | Any tap gives visual feedback within 50 ms; animations hold 60 fps on mid-range devices. |
| NFR-PERF-004 | API p95 < 200 ms for reads, < 400 ms for order placement (server time). |
| NFR-PERF-005 | New order visible on KDS/board < 2 s p95 after server accepts it. |
| NFR-REL-001 | 99.95 % availability during outlet service hours. |
| NFR-REL-002 | Zero lost or duplicated orders (idempotency + outbox, doc 09). |
| NFR-REL-003 | Guest app keeps the loaded menu and cart usable with no connection; KDS/board survive disconnects and resync. |
| NFR-SCL-001 | Architecture supports 5,000 outlets, 100,000 concurrent guests, 200 orders/s burst, without redesign (doc 08 §9). |
| NFR-SEC-001 | Strict tenant isolation enforced in application code and database Row-Level Security. |
| NFR-SEC-002 | All traffic over HTTPS (TLS 1.2+), HSTS; secrets never in code. |
| NFR-PRV-001 | Compliance with India's DPDP Act 2023: notice, consent, purpose limitation, deletion, breach process (doc 10 §8). |
| NFR-A11Y-001 | WCAG 2.2 AA for guest and admin apps; respects reduced-motion and large-text settings. |
| NFR-I18N-001 | All UI strings externalised in English and Hindi; menu content fields have optional Hindi values; INR and dates formatted in Indian style (`₹1,25,000`, `12 Mar, 8:42 PM`), with Devanagari rendering tested on every guest screen. |
| NFR-BRW-001 | Guest app supports Android Chrome / Samsung Internet (last 3 years), iOS Safari 15+, in-app browsers of popular QR scanner and camera apps. Admin supports latest Chrome, Edge, Safari, Firefox. |

---

## 7. Status models (product view)

These define what guests and staff see. The technical state machines (with allowed transitions) are in doc 06 §7.

**Order**: `Sent (placed)` → `Confirmed` → `Preparing` → `Ready` → `Served`; side exits: `Declined` (by staff before confirm), `Cancelled` (by guest before preparing, or staff any time with reason).

**Table session**: `Open` → `Bill requested` → `Settled` → `Closed`; or `Open` → `Closed (abandoned / timed out)`; `Merged` (into another session).

**Bill**: `Open` → `Partially paid` → `Paid` → `Settled`; `Settled` → `Voided` (with reversal).

**Service request**: `Open` → `Acknowledged` → `Done`; `Cancelled`.

---

## 8. Notifications matrix

| Event | Guest | Board / staff app | KDS | Owner |
|-------|-------|-------------------|-----|-------|
| Order placed | Status screen | Sound + card | Ticket (after confirm, or immediately if auto) | — |
| Order confirmed/declined | Status update | — | Ticket appears | — |
| Order ready | Status + optional vibration | Card moves | Ticket bumped | — |
| Service request | "On the way" after ack | Sound + panel | — | — |
| Bill requested | Status | Sound + spot highlight | — | — |
| Order waiting > threshold | — | Card turns amber/red | Ticket colour | — |
| Low rating (P1) | — | Manager notification | — | Notification |
| Day close | — | — | — | Summary (P1) |

---

## 9. Plans, features, and limits (founder-defined — D-13)

The founder sets all prices and decides what each plan contains, using the plan builder (PLT-003). The software only provides the switches and limits below; it ships with **no built-in plans**.

**Feature flags available to the plan builder** (each can be on/off per plan and overridden per tenant):

| Flag key | Feature |
|----------|---------|
| `ordering` | QR ordering (off = view-only digital menu) |
| `pay_at_counter` | Pay at counter flow |
| `online_payments` | Online payment plug-in (Phase 8) |
| `kds` | Kitchen display screens |
| `kitchen_stations` | Routing items to stations |
| `printing` | Thermal printer KOT / bill printing |
| `customer_recognition` | Guest OTP identity, history, reorder |
| `customer_export` | Customer CSV export |
| `insights_basic` | Basic insight cards |
| `insights_full` | Full insight library and daily summary |
| `reports_export` | CSV / XLSX / PDF report exports |
| `coupons` | Coupons and offers |
| `hindi_menu` | Hindi switch for guests |
| `custom_roles` | Custom staff roles |
| `shared_cart` | Shared table cart |
| `feedback` | Guest ratings and comments |
| `room_service` | Room spots, room-service menu, charge to room |
| `multi_outlet` | More than one outlet per tenant |

**Limits** (numbers per plan, overridable per tenant): `max_outlets`, `max_spots_per_outlet`, `max_staff`, `max_kds_devices`, `max_menu_items`, `max_photo_storage_mb`.

When a tenant reaches a limit, the admin shows "You've reached your plan's limit of 20 tables. Contact us to upgrade." (doc 09 `PLAN_LIMIT_REACHED`). When a feature is off, its menu entries are hidden or shown with a lock and the same message. Data is never deleted when a plan is downgraded — only the feature becomes inaccessible.

---

## 10. Future ideas (not committed)

Table reservations and waitlist · loyalty points · gift cards · delivery aggregator sync · ingredient inventory with auto sold-out · PMS integration for hotels (charge to room automatically) · digital signage menu boards · AI menu photography enhancement · dynamic pricing / happy hours by demand · multi-brand cloud kitchen support · customer-facing order screen ("now serving") for pickup counters.

---

## 11. Acceptance checklist for the first sellable release (P0)

- [ ] A new tenant can be created, branded, given a menu with photos, spots, and printed QR cards without developer help.
- [ ] A guest can scan, browse, customise, order, follow status, add a second round, request bill, and be settled at counter.
- [ ] Orders are never lost or duplicated under the network scenarios in doc 13 §5.
- [ ] Returning verified guests are recognised and can reorder.
- [ ] Kitchen and floor staff can run a full dinner service on KDS and the orders board.
- [ ] Owner sees today's numbers, insight cards, and can export GST and sales reports.
- [ ] Roles restrict actions as in doc 10 §3; audit log records sensitive actions.
- [ ] Performance budgets in doc 08 are met on the reference devices.
