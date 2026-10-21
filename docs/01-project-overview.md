# 01 — Project Overview

## 1. One-line summary

A premium, multi-tenant QR ordering platform: guests scan a code at their table or room, order from a beautiful menu, and pay; businesses control everything themselves from an intelligent admin panel.

## 2. The problem

| Who | Today's pain |
|-----|--------------|
| **Guests** | Wait to get a waiter's attention, wait for a paper menu, wait to order, wait for the bill. Menus are often stained PDFs with no photos. Nobody remembers what they liked last time. |
| **Restaurant / cafe / hotel owners** | Orders get misheard, menu changes need reprinting, there is no record of who their customers are, and reports (if any) are raw numbers nobody has time to read. Existing QR products look cheap, are slow on crowded Wi-Fi, and need the vendor to change anything. |
| **Staff** | Running between tables to take orders, shouting orders to the kitchen, chasing bills. |

## 3. The vision

The finest ordering experience in hospitality:

- **For guests**: opening the menu feels instant, browsing feels like a premium magazine, ordering takes seconds, and the place remembers them.
- **For owners**: every control is in their hands — dishes, photos, prices, tables, QR codes, staff, taxes, branding — and the dashboard *tells* them what is happening instead of showing raw data.
- **For us (the platform)**: onboard a new business in minutes and never be called to "change a price".

### Product principles

1. **The guest never waits.** Every tap responds instantly; the network catches up in the background.
2. **Nothing is ever lost.** An order placed on a weak connection reaches the kitchen exactly once.
3. **Owners need no developer.** If an owner has to call us to change something, that is a missing feature.
4. **Calm, premium, minimal.** Fewer elements, better made. Motion explains, never decorates.
5. **Insights over data.** The dashboard leads with sentences and actions; numbers are one tap deeper.
6. **Private by default.** Each business owns its customers; guests control their data.

## 4. Users

| User | Description | Main device |
|------|-------------|-------------|
| **Guest** | Person at a table or in a hotel room. Any age, any tech comfort, often on a weak connection. | Their own phone (Android majority, iOS) |
| **Owner** | Owns the business account. Full control including billing and staff. | Laptop + phone |
| **Manager** | Runs daily operations: menu availability, orders, staff shifts, reports. | Laptop / tablet / phone |
| **Cashier** | Settles bills at the counter, applies discounts within limits. | Counter tablet / PC |
| **Waiter / captain** | Watches table requests, places orders on behalf of guests, marks orders served. | Phone |
| **Kitchen staff** | Sees incoming orders, marks them preparing/ready. | Wall-mounted tablet (KDS) |
| **Platform admin (you)** | Onboards businesses, manages plans, supports customers. | Laptop |

## 5. Scope

### In scope (full product)

- Guest web app: QR entry, menu, item details, customisation, cart, ordering, live order status, running table bill, service requests ("call waiter", "request bill"), guest identity with OTP, order history and reorder, pay at counter, online payment (via plug-in, final phase).
- Admin web app: menu studio, branding, outlets, spots (tables/rooms) and QR codes, live orders board, kitchen display, billing and settlement, discounts, taxes, staff and roles, customers, dashboard with insights, reports and exports, settings, audit log.
- Platform admin: tenant onboarding, plans and feature flags, support tools, system health.
- Infrastructure: multi-tenant backend, realtime, background jobs, image pipeline, monitoring, backups.

### Out of scope (for now)

- Building a payment gateway (we plug in the founder's existing one — D-03).
- Inventory/stock management by ingredient, purchasing, supplier management.
- Delivery aggregator integrations (Swiggy, Zomato), table reservations, loyalty points wallets. These are listed as future ideas in doc 02 §10.
- Native mobile apps.

## 6. Business model

- **One shared platform (multi-tenant SaaS), sold as a subscription** (D-01). Every business gets its own private space on the same system: its own menu, staff, customers, reports, and branding. No business can see another's data.
- **Why this is the easiest model to run and sell:**
  - Selling to a new restaurant takes minutes: create the business in the platform admin, send the owner an invite, print the QR cards. Nothing is installed anywhere.
  - One codebase and one set of servers for every customer. A bug fix or new feature reaches all businesses the moment it ships.
  - The founder sees all customers, their plans, usage, and health in one panel, and can suspend, upgrade, or help any business from there.
  - Costs grow slowly: 10 or 1,000 businesses run on the same infrastructure, scaled up as needed (doc 14).
- **Market**: India only (D-02) — INR, GST, UPI, +91 numbers. Guest app in English with a Hindi switch (D-11).
- **Pricing and plans are decided by the founder** and configured in the plan builder (doc 02 §9, PLT-003). The software has no fixed prices.
- The founder sets up each business (or the business self-onboards later), prints QR cards, and hands over.
- Online payment fees are between the business and the payment gateway; the platform can optionally take a small platform fee later (not in first release).

## 7. Success metrics

| Metric | Target |
|--------|--------|
| Time from QR scan to menu visible (4G, mid-range Android) | < 1.5 s (p75) |
| Time from QR scan to order placed (first-time guest, 2 items) | < 60 s median |
| Order delivery to kitchen after "Place order" | < 2 s (p95), exactly once |
| Orders placed through QR vs. via staff | > 60 % after 1 month at a venue |
| Returning guest recognition rate (same device or verified phone) | > 40 % of repeat visits |
| Owner self-service | 0 support tickets for menu/price/QR changes |
| Platform uptime during service hours | 99.95 % |
| Business retention after 6 months | > 90 % |

## 8. High-level system picture

```
 Guest phone ──(QR link)──▶ CDN ──▶ Guest Web App (Next.js)
                                  │
 Admin / Staff / KDS ──────────▶ Admin Web App (Next.js)
                                  │
                                  ▼
                       API (Fastify, modular monolith)
                      ╱        │          │         ╲
               PostgreSQL    Redis    Object storage  Realtime gateway
                               │       (photos)       (Socket.IO)
                            Worker (jobs: images, insights, SMS, exports)
                                  │
                     Plug-in: payment gateway (final phase)
```

Full detail in [05-architecture.md](./05-architecture.md).

## 9. Glossary

| Term | Meaning |
|------|---------|
| **Platform** | Our whole system, serving all businesses. |
| **Tenant** | One customer business account (e.g. "Blue Tokai Group"). All its data is isolated from other tenants. |
| **Outlet** | One physical location of a tenant (e.g. "Blue Tokai — Bandra"). Menus, spots, orders, and staff assignments are per outlet. |
| **Area** | A section of an outlet (e.g. "Rooftop", "Ground floor", "Floor 3 rooms"). |
| **Spot** | Any place a guest can order from: a table, a hotel room, a counter seat, a poolside cabana. Each spot has its own QR code. |
| **QR code** | A printed code containing a short link with an unguessable token that identifies one spot. |
| **Table session** | The period from the first order at a spot until its bill is settled and the spot is cleared. All orders in a session share one bill. |
| **Order** | One submission from a guest (or waiter) — a "round". A session can have many orders. In kitchens this is a KOT (Kitchen Order Ticket). |
| **Bill** | The total for a table session: items, discounts, taxes, service charge, rounding, payments. |
| **Item** | A dish or drink on the menu. |
| **Variant** | A size or form of an item with its own price (Half / Full, Regular / Large). |
| **Modifier group / option** | Customisations (e.g. "Choose milk": Oat +₹40, Almond +₹50). Groups have min/max selection rules. |
| **Menu** | A set of categories and items available at certain times (e.g. "Breakfast 7–11 AM"). |
| **Menu snapshot** | A frozen, versioned copy of everything a guest needs to render the menu, cached at the CDN. |
| **KDS** | Kitchen Display System — the kitchen's tablet screen showing orders. |
| **Customer** | A guest known to a tenant through a verified phone number. |
| **Guest device** | An anonymous browser identity (cookie) before the guest gives their phone number. |
| **Business day** | The reporting day for an outlet, which can end after midnight (e.g. 4:00 AM cutoff). |
| **Insight** | A plain-language observation on the dashboard with an explanation and a suggested action. |
| **Feature flag** | A switch that enables a feature for a tenant, based on its plan or a manual override. |
| **Minor units** | The smallest unit of a currency (paise for INR). All money is stored this way. |
