# 00 — Decisions & Assumptions Log

Every important decision is recorded here with a reason. Items marked **Assumed** were decided on the founder's behalf using the recommended option and can be changed; changing one means updating the documents listed in its "Affects" column.

Status values: **Decided** (founder confirmed), **Assumed** (recommended default, awaiting confirmation), **Open** (needs an answer before the phase that depends on it).

## Product decisions

| ID | Decision | Status | Reason | Affects |
|----|----------|--------|--------|---------|
| D-01 | The product is a **multi-tenant SaaS**: one platform, many businesses, each with its own isolated space. Businesses pay a subscription. The founder runs everything (creating businesses, plans, suspensions, support) from one **platform admin panel**. | Decided | The founder chose the model that is easiest to manage and sell at scale. Bugs are fixed once, features ship to every customer at the same time, and there is one system to run. Selling to a new business means creating an account (minutes), not installing anything. A separate install per business would multiply maintenance work with every new customer. | 01, 05 |
| D-02 | The product is built **for India only**: INR, GST, UPI, +91 phone numbers, Asia/Kolkata time zone, Indian number formatting (₹1,25,000), FSSAI dietary marks. Currency and time zone are still stored per outlet so the data model stays clean, but the UI does not offer other countries. | Decided | Founder instruction. | 02, 05 |
| D-03 | **Payment gateway is out of scope until the final phase.** The founder already has a working Razorpay-style integrator. We design only a clean plug-in point (doc 05 §16) and build "Pay at counter" fully. | Decided | Founder instruction. | 02, 05 |
| D-04 | Guests use a **web app opened from the QR code**. No app download. | Assumed | Any download step loses most guests at the table. | 03, 05 |
| D-05 | Guests identify with **phone number verified by OTP** (SMS to +91 numbers; WhatsApp later). The OTP requirement can be switched off per business. | Decided | Founder confirmed. Without verification, anyone could type another person's number and see their order history. | 02, 05 |
| D-06 | Guest details are asked **at checkout, not before browsing**. Returning guests on the same device are recognised automatically. | Assumed | Browsing must have zero friction. | 02, 04 |
| D-07 | **Customer records belong to each business.** A guest who visits Cafe A and Cafe B has two separate customer records, one owned by each business. Businesses never see each other's customers. | Assumed | Privacy, trust with business owners, and India's Digital Personal Data Protection Act (DPDP) 2023. | 05 |
| D-08 | Supports **tables and hotel rooms** (and other spots such as poolside or counters) through one generic concept: a **Spot**. | Assumed | Same flow, one implementation. | 02, 05 |
| D-09 | **Shared table cart** (several phones adding to one cart) ships in a later phase; in the first release each phone has its own cart, but all orders at a table land on one **shared table bill**. | Assumed | Big wow factor but complex; the shared bill already covers the main need. | 02, 05 |
| D-10 | **Kitchen Display (KDS) on a tablet** in the first release. Thermal printer support (KOT printing) in a later phase through a small print-bridge app. | Assumed | Tablets need no hardware integration; printers are common in India, so they come next. | 02, 05 |
| D-11 | **English is the default language; guests can switch to Hindi** with one tap. All guest app text ships in both languages, and businesses can add Hindi names and descriptions for their dishes (falling back to English where missing). The admin app is English, with Hindi as a later option. No other languages. | Decided | Founder instruction. | 02, 03, 05 |
| D-12 | Insights start **rule-based** (reliable, explainable, cheap). AI-written summaries come later as an optional layer over the same data. | Assumed | Rules never invent numbers; AI wording can be added safely on top. | 05 |
| D-13 | **The founder decides all pricing and plans.** The software provides a **plan builder** in the platform admin: create any number of plans, set their price and billing period, and choose which features and limits each includes. Any tenant can also get individual overrides. Billing businesses is manual (invoice + bank transfer) in the first release. | Decided | Founder instruction. Nothing about pricing is hard-coded, so plans can change without a developer. | 02, 05 |
| D-14 | First order of a new table session can require **staff confirmation** (default on). Later orders in the same session go straight to the kitchen. Configurable per business. | Assumed | Stops prank orders from a photographed or shared QR without slowing real guests. | 02, 05 |
| D-15 | **Service charge is always optional for the guest** (shown as a removable line), in line with India's Central Consumer Protection Authority 2022 guidelines. | Assumed | Legal safety for our customers. | 02, 05 |
| D-16 | Working product name is **"Hotel"**. Final brand name is open. | Open | Needed before public launch, not before building. | 01, 03 |
| D-17 | The only personal detail asked from guests is their **name** (plus the phone number used for OTP). No email, birthday, or other fields. | Decided | Founder instruction. Fewer fields means faster checkout and less personal data to protect. | 02, 04, 05 |
| D-18 | Documentation is limited to **six documents** (00–05). Database, API, error handling, security, performance, insights, payment plug-in point, and build phases all live inside doc 05. No QA, testing-process, or Docker/deployment planning documents. | Decided | Founder instruction. | All |

## Technical decisions

| ID | Decision | Status | Reason |
|----|----------|--------|--------|
| T-01 | **TypeScript everywhere**, in one monorepo (pnpm workspaces + Turborepo). | Assumed | One language, shared types and validation between apps. |
| T-02 | **Next.js (React)** for the guest app and the admin app (separate apps). | Assumed | One UI stack; server rendering gives fast first paint on slow networks. Strict JavaScript budgets (doc 05 §12) keep the guest app light. |
| T-03 | **Node.js + Fastify** API as a **modular monolith**, plus a separate **worker** process for background jobs. | Assumed | Fastest path with clean module boundaries; modules can be split into services later if ever needed. |
| T-04 | **PostgreSQL** as the only primary database, with `tenant_id` on every business table and **Row-Level Security** as a second safety net. | Assumed | Relational data, strong consistency for money, proven at scale. |
| T-05 | **Redis** for caching, rate limiting, OTP codes, job queues (BullMQ), and realtime fan-out. | Assumed | One fast in-memory store for all short-lived data. |
| T-06 | **Realtime via Socket.IO** (WebSocket with automatic long-polling fallback). Realtime is an optimisation only; every screen can rebuild its state from REST after a disconnect. | Assumed | Works behind bad networks and proxies; never loses an order because a socket dropped. |
| T-07 | **Menu is served as a versioned, immutable snapshot** cached at the CDN. A tiny pointer endpoint says which version is current. | Assumed | Menu loads come from a CDN close to the guest even when a venue is packed; no cache purging needed. |
| T-08 | **Images** stored in S3-compatible storage (Cloudflare R2), resized by the worker into AVIF/WebP at fixed sizes with tiny blurred placeholders. | Assumed | Photos are the heaviest part of a menu; this keeps them fast. |
| T-09 | **Money stored as integers in minor units** (paise). Never floating point. | Decided (engineering rule) | Avoids rounding errors in bills. |
| T-10 | Runs on **managed cloud services in Mumbai** (managed PostgreSQL, managed Redis, app hosting) behind **Cloudflare** CDN. | Assumed | Low latency for Indian venues, data stays in India, no servers to maintain by hand. |
| T-11 | **Drizzle ORM** with plain SQL migrations. | Assumed | Type-safe, close to SQL, fast. |
| T-12 | **Zod** schemas in a shared package are the single definition of every API request/response; OpenAPI is generated from them. | Assumed | One source of truth for validation and API reference. |

## Open questions (answer before the listed phase)

Phases are defined in doc 05 §20.

| ID | Question | Needed by |
|----|----------|-----------|
| Q-01 | Final brand name and domain (for QR short links, e.g. `hotl.in`). | Phase 1 (QR printing) |
| Q-02 | SMS provider account for OTP (MSG91 recommended for India; DLT template registration with telecom operators takes 1–2 weeks, start early). | Phase 4 |
| Q-04 | Interface of the founder's existing payment integrator, to map onto doc 05 §16. | Phase 8 |
| Q-05 | Which thermal printer models pilot customers use. | Phase 9 |
