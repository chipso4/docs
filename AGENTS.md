# Continuum Integration Documentation

## About this project

- Documentation site built on [Mintlify](https://mintlify.com) for Continuum's Manufacturer Warranty Hub integration
- Pages are MDX files with YAML frontmatter
- Configuration lives in `docs.json`
- Run `mint dev` to preview locally
- Run `mint broken-links` to check links

## What this documents

This is a **reverse API** documentation portal. It documents the API contract that **manufacturers must implement** so that Continuum can call their middleware. Continuum acts as the consumer-facing warranty portal; the manufacturer builds the middleware API layer between Continuum and their systems of record.

## Audience

Partner/customer developers and IT teams who need to build the middleware API.

## Terminology

- **Continuum** — the warranty automation platform (us)
- **Manufacturer** — the company building the middleware (the reader)
- **Middleware API** — the API layer the manufacturer implements; Continuum calls it
- **Systems of record** — the manufacturer's ERP, warranty database, legacy systems
- **Unit** — a manufactured product identified by serial number
- **Coverage tier** — a specific category of warranty coverage (tank, parts, labor)
- **Entitlement** — the determination of what's covered for a specific serial number
- **BOM** — bill of materials; the list of serviceable parts in a unit
- **Cross-reference** — alternate identifiers for a part (superseded numbers, brand equivalents)
- **Media** — documents, images, manuals, diagrams — all served through a unified endpoint
- **extendedAttributes** — the generic name-value pair mechanism for industry-specific data

## Style preferences

- Use active voice and second person ("you" = the manufacturer's IT team)
- Keep sentences concise — one idea per sentence
- Use sentence case for headings
- Bold for UI elements: Click **Settings**
- Code formatting for file names, commands, paths, endpoints, and field names
- JSON examples should be realistic (use water heater data as the canonical example)
- Every endpoint page should include a "Used in" section linking to the process flow(s) that use it
- Cross-link generously between endpoint pages, data type pages, and process flows

## Content boundaries

- DO document: manufacturer-facing middleware endpoints, data types, process flows, implementation guidance
- DO NOT document: Continuum's internal claims endpoints, Continuum UI details, Continuum's internal architecture
- Reference Continuum-managed endpoints only where needed for process flow context (e.g., claim statuses)

## Structure

- `/overview/` — what, why, architecture, key concepts
- `/getting-started/` — implementation guide, auth, error handling
- `/process-flows/` — end-to-end workflow descriptions with sequence diagrams
- `/api-reference/` — individual endpoint specs organized by domain
- `/data-types/` — shared data type reference
- `/specs/` — internal working documents (not published)
