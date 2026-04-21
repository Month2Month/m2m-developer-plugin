# M2M Repository Map

Canonical guide to what each Month2Month repo does, so you can decide which repos are plausibly in scope for a given feature before querying PRs. Sourced from the p2p workspace architecture doc.

This is **not exhaustive** — the `Month2Month` org has more repos than are documented here. For any repo you encounter that isn't in this file, fall back to `gh repo view Month2Month/<repo> --json description,name`. New repos get added over time; this file is a starting point, not a closed list.

## Core backend (authoritative business logic)

| Repo | Stack | Role | Owns |
|---|---|---|---|
| `holidale-web` | Ruby on Rails (monolith) | PMS system of record | Properties, inventory, bookings, tenants, ops workflows, core APIs. Also still serves some legacy PMS UI pages (Rails views). |
| `quotation_center` | Ruby on Rails (service) | Pricing authority | All pricing rules, quote generation, breakdowns, rate/fee/discount/markup logic. |
| `financial_god` | Ruby on Rails (service) | Finance authority | Owner revenue, payouts, settlements, financial approvals, audit trails, Lark approval workflow. |

## Web frontends (presentation only — no business logic)

| Repo | Stack | URL | Audience |
|---|---|---|---|
| `m2m_pms_frontend` | Next.js | internal.month2month.com | Internal staff. Consumes holidale-web APIs. Partial migration — some PMS pages still rendered by holidale-web Rails views. |
| `spark-ssr` | React SSR | month2month.com | Public site / prospective tenants. Consumes holidale-web APIs only. |

## AI & automation

| Repo | Stack | Role |
|---|---|---|
| `mia-m2m-mcp` | Python | AI gateway / MCP server — enforces permissions, converts AI intent into backend actions. AI features should route through here when possible. |
| `mia-chatbot-api` | (TBD) | Month2Month's AI housing search assistant. |

## Mobile apps (thin clients)

| Repo | Stack | Audience |
|---|---|---|
| `m2m_business_app` | React Native | Vendors — field tasks, submissions, checklists. |
| `m2m-taro3-miniapp` | Taro3 (WeChat) | Homeowners — view bookings, revenue, property status. |

## Domain → likely repos lookup

When the user's feature description points at a domain, these are the repos to prioritize. Use this as a quick triage — the actual PR search still tells the truth.

| Domain in the feature | Likely repos |
|---|---|
| Booking lifecycle, properties, inventory, tenant data | `holidale-web`, plus `m2m_pms_frontend` / `spark-ssr` for UI |
| Pricing, rates, discounts, quotes, ADR, markups | `quotation_center`, plus `spark-ssr` if prices are shown publicly |
| Owner revenue, payouts, settlements, approvals | `financial_god` |
| Internal staff tool / PMS dashboard UX | `m2m_pms_frontend` (new pages) or `holidale-web` (legacy pages) |
| Public-facing listing, search, booking submission | `spark-ssr` |
| AI features, automation, agent tools | `mia-m2m-mcp`, `mia-chatbot-api` |
| Vendor-facing mobile workflows | `m2m_business_app` |
| Homeowner mobile (WeChat) | `m2m-taro3-miniapp` |

## Things to watch for

- A cross-cutting feature (e.g. a new pricing surface) may touch pricing logic in `quotation_center`, expose an API in `holidale-web`, and surface in `spark-ssr` and/or `m2m_pms_frontend` — expect to pull PRs from 3–4 repos.
- Features that sound purely like UI often have a backend half in `holidale-web` (API change) that the user may not think to mention — check it even if the feature description only names the frontend.
- Finance-adjacent changes almost always touch `financial_god` even if the visible change is elsewhere.
