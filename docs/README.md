# Hotel — Product & Engineering Documentation

> **Hotel** is the working name of a multi-tenant QR ordering platform for restaurants, cafes, and hotels. Guests scan a QR code on the table (or in a room), browse a beautiful menu, order, and pay online or at the counter. Each business runs everything from its own admin panel without needing a developer.

This folder is the single source of truth for what we are building and how. Build the product **phase by phase** using [15-development-roadmap.md](./15-development-roadmap.md); every phase links back to the sections of the other documents it depends on.

## Reading order

| # | Document | What it answers |
|---|----------|-----------------|
| 00 | [Decisions & assumptions log](./00-decisions-log.md) | Which product/technical decisions were made, why, and which are still open |
| 01 | [Project overview](./01-project-overview.md) | Vision, users, scope, business model, glossary |
| 02 | [Product requirements (PRD)](./02-prd.md) | Every feature, with requirements and acceptance criteria |
| 03 | [Design system (DRD)](./03-design-system.md) | Visual language, tokens, motion, components, tenant theming, accessibility |
| 04 | [User flows & screens](./04-user-flows-and-screens.md) | Step-by-step flows and every screen with its states |
| 05 | [Architecture](./05-architecture.md) | System components, tech stack, multi-tenancy, data flow, scaling |
| 06 | [Database](./06-database.md) | Every table, column, relation, index, and data rule |
| 07 | [API specification](./07-api-spec.md) | REST endpoints, realtime events, conventions |
| 08 | [Performance & reliability](./08-performance-reliability.md) | Speed budgets, slow-network and offline strategy, load targets |
| 09 | [Error handling](./09-error-handling.md) | Error taxonomy, codes, retries, idempotency, failure scenarios, UX copy |
| 10 | [Access control & security](./10-access-control-security.md) | Roles, permissions matrix, auth, tenant isolation, privacy/DPDP |
| 11 | [Insights engine](./11-insights-engine.md) | Dashboard metrics and the plain-language insight rules |
| 12 | [Payment integration contract](./12-payment-integration-contract.md) | The plug-in interface the existing payment gateway attaches to |
| 13 | [Testing & QA](./13-testing-qa.md) | Test strategy, device/network matrix, release checklist |
| 14 | [DevOps & deployment](./14-devops-deployment.md) | Environments, CI/CD, infrastructure, monitoring, backups |
| 15 | [Development roadmap](./15-development-roadmap.md) | Phases, deliverables, and exit criteria |
| 16 | [Engineering conventions](./16-engineering-conventions.md) | Repo layout, code style, naming, commits, reviews |

## How to use these documents

1. Read **00 → 02** to understand the product.
2. Before building any phase, read the phase in **15** and every section it references.
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
