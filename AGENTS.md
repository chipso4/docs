# Continuum Integration Documentation

## About this project

- Documentation site built on [Mintlify](https://mintlify.com) for Continuum's integration platform
- Pages are MDX files with YAML frontmatter
- Configuration lives in `docs.json`
- Run `mint dev` to preview locally
- Run `mint broken-links` to check links

## What this documents

This portal documents **three integration hubs** — the API contracts that partners must implement so Continuum can connect to their systems. Each hub is a reverse integration: the partner builds the API, Continuum calls it.

### The three hubs

| Hub | Product tag | Audience | Direction |
|-----|-------------|----------|-----------|
| **Returns Hub** (`btuReturns`) | Distributor IT teams | Continuum → Distributor ERP |
| **Warranty Hub** (`btuWarranties`) | Distributor IT teams | Continuum → Distributor ERP |
| **Warranty Claims Hub** (`btuWarrantyVendor`) | Manufacturer IT teams | Continuum → Manufacturer middleware |

The Returns and Warranty hubs share a common foundation (customers, products, invoices). The Claims Hub is independent — it connects to the OEM, not the distributor's ERP.

## Audience

- **Returns Hub / Warranty Hub** — Distributor developers and IT teams building the ERP API layer
- **Warranty Claims Hub** — Manufacturer developers and IT teams building the middleware API layer

## Terminology

- **Continuum** — the warranty and returns automation platform (us)
- **Distributor** — the company selling products and processing returns (builds the ERP API)
- **Manufacturer / OEM** — the company that makes the products and processes warranty claims (builds the middleware API)
- **Hub** — a self-contained integration product with its own endpoints, flows, and implementation checklist
- **Unit** — a manufactured product identified by serial number
- **RMA** — Return Merchandise Authorization; the document authorizing a product return
- **Coverage tier** — a specific category of warranty coverage (tank, parts, labor)
- **Entitlement** — the determination of what's covered for a specific serial number
- **BOM** — bill of materials; the list of serviceable parts in a unit
- **Cross-reference** — alternate identifiers for a part (superseded numbers, brand equivalents)
- **Media** — documents, images, manuals, diagrams — all served through a unified endpoint
- **extendedAttributes** — the generic name-value pair mechanism for industry-specific data
- **V1 / V2** — API version generations. V1 uses operation-per-route PATCH endpoints. V2 uses entity-focused REST with unified Core types (CoreOrganization, CoreRma, CoreOrder, etc.)

## Style preferences

- Use active voice and second person ("you" = the partner's IT team)
- Keep sentences concise — one idea per sentence
- Use sentence case for headings
- Bold for UI elements: Click **Settings**
- Code formatting for file names, commands, paths, endpoints, and field names
- JSON examples should be realistic (use water heater / HVAC data as the canonical example)
- Every endpoint page should include a "Used in" section linking to the hub(s) and process flow(s) that use it
- Cross-link generously between hub pages, endpoint pages, data type pages, and process flows
- Each hub tab should be self-contained — a customer implementing only Returns should never need to visit the Warranty Claims tab

## Content boundaries

- DO document: all three hub integration contracts, shared data types, process flows, implementation guidance
- DO NOT document: Continuum's internal endpoints, Continuum UI details, Continuum's internal architecture
- Reference Continuum-managed behavior only where needed for process flow context (e.g., claim statuses, invoice caching)

## Structure

```
/                          — Landing page (all 3 hubs)
/overview/                 — What, why, architecture, key concepts
/getting-started/          — Implementation guide, auth, error handling
/returns-hub/              — Returns Hub (self-contained)
  introduction.mdx
  return-flow.mdx
  implementation-checklist.mdx
/warranty-hub/             — Warranty Hub (self-contained)
  introduction.mdx
  warranty-flow.mdx         — The 5-phase orchestration
  replacement-orders.mdx
  vendor-returns.mdx
  implementation-checklist.mdx
/warranty-claims-hub/      — Warranty Claims Hub (self-contained)
  introduction.mdx
  claim-flow.mdx
  status-lifecycle.mdx
  implementation-checklist.mdx
/process-flows/            — Detailed process flow pages (Claims Hub context)
/api-reference/            — Unified endpoint reference (all hubs, with hub badges)
/data-types/               — Shared data type reference (migrating to V2 Core types)
/specs/                    — Internal working documents (not published)
```

## Navigation (docs.json tabs)

1. **Overview** — Introduction, architecture, key concepts, getting started
2. **Returns Hub** — Self-contained guide for `btuReturns`
3. **Warranty Hub** — Self-contained guide for `btuWarranties`
4. **Warranty Claims Hub** — Self-contained guide for `btuWarrantyVendor` + existing process flows
5. **API Reference** — All endpoints unified, with hub ownership badges + data types
