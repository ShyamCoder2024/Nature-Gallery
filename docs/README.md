# Hotel — Product & Engineering Documentation

> **Hotel** is the working name of a multi-tenant QR ordering platform for restaurants, cafes, and hotels. Guests scan a QR code on the table (or in a room), browse a beautiful menu, order, and pay online or at the counter. Each business runs everything from its own admin panel without needing a developer.

This folder is the single source of truth for what we are building and how. There are six documents. Build the product **phase by phase** using the build plan in [05-architecture.md §20](./05-architecture.md#20-build-phases); every phase links back to the requirements and sections it depends on.

## Reading order

| # | Document | What it answers |
|---|----------|-----------------|
| 00 | [Decisions & assumptions log](./00-decisions-log.md) | Which product/technical decisions were made, why, and which are still open |
| 01 | [Project overview](./01-project-overview.md) | Vision, users, scope, business model, glossary |
| 02 | [Product requirements (PRD)](./02-prd.md) | Every feature, with requirements and acceptance criteria |
| 03 | [Design system (DRD)](./03-design-system.md) | Visual language, tokens, motion, components, tenant theming, accessibility |
| 04 | [User flows & screens](./04-user-flows-and-screens.md) | Step-by-step flows and every screen with its states |
| 05 | [Architecture](./05-architecture.md) | The complete engineering design: tech stack, code structure, multi-tenancy, database schema, API, menu caching, order pipeline, realtime, performance and offline, error handling, security and roles, insights engine, payment plug-in point, build phases |

## How to use these documents

1. Read **00 → 02** to understand the product.
2. Before building any phase, read the phase in **05 §20** and every requirement and section it references.
3. When a decision changes, update **00-decisions-log.md** first, then the affected documents. Documents must never contradict each other; if they do, the decisions log wins.
4. Requirement IDs (for example `GST-ORD-004`) are stable. Reference them in tickets, commits, and tests.

## Requirement ID scheme

| Prefix | Area |
|--------|------|
| `GST-*` | Guest experience (`GST-MENU`, `GST-CART`, `GST-ORD`, `GST-ID`, `GST-PAY`, `GST-SRV`) |
| `ADM-*` | Business admin (`ADM-MENU`, `ADM-TBL`, `ADM-ORD`, `ADM-STF`, `ADM-BRD`, `ADM-CUS`, `ADM-DSH`, `ADM-SET`, `ADM-BIL`) |
| `KDS-*` | Kitchen display |
| `PLT-*` | Platform admin (you, the seller) |
| `NFR-*` | Non-functional requirements (performance, reliability, security, accessibility) |
