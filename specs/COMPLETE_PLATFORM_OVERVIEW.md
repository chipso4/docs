# Manufacturer Warranty Hub — Complete Platform Overview

**Status:** Draft  
**Last Updated:** March 29, 2026  
**Purpose:** Complete picture of the Manufacturer Warranty Hub — how it works end-to-end, what APIs exist on both sides, and what's still open.

---

## 1. What Is This?

Continuum is a warranty and returns automation platform. For the Manufacturer Warranty Hub, Continuum operates as the **consumer-facing warranty portal** on behalf of the manufacturer. Consumers (homeowners), contractors, and distributors interact with Continuum's UI. Behind the scenes, Continuum calls into the manufacturer's middleware API to retrieve data from and push updates to their systems of record (ERPs, warranty databases, legacy systems — often green-screen terminals where manual data entry is the norm).

This document captures the **complete platform** — both sides of the integration:

- **Manufacturer endpoints** — What the manufacturer builds. Continuum calls these to look up data and sync registrations.
- **Continuum endpoints** — What Continuum owns internally. Claims lifecycle, attachments, status tracking, and claim history.

### 1.1 Architecture

```
Consumer / Contractor / Distributor
        │
        ▼
   ┌─────────────┐          ┌──────────────────────┐          ┌──────────────┐
   │  Continuum   │──HTTPS──▶│  Manufacturer's       │─────────▶│  ERP /       │
   │  Platform    │◀─────────│  Middleware API        │◀─────────│  Warranty DB │
   └─────────────┘          └──────────────────────┘          └──────────────┘
    (Hosts portal,            (Manufacturer builds              (SAP, Oracle,
     claims engine,            and hosts this)                   NetSuite, etc.)
     attachments)
```

### 1.2 Key Actors

| Actor | Role |
|-------|------|
| **Consumer / Homeowner** | Registers products, checks warranty, initiates service requests |
| **Contractor** | Diagnoses failures, files warranty claims, selects replacement parts |
| **Distributor** | Sells products, processes warranty exchanges at the counter |
| **Manufacturer** | Owns warranty terms, parts inventory, claims adjudication. Builds the middleware API. |
| **Continuum** | Orchestrates the process. Provides the UI, claims engine, and automation. Calls the manufacturer's API. |

---

## 2. End-to-End Process Flows

### 2.1 Product Registration

Consumer registers their product after purchase/installation.

1. Consumer enters serial number on the Continuum portal.
2. Continuum calls the manufacturer API to **validate the serial** and **retrieve unit details** (model, product info, manufacturing data).
3. Portal auto-fills product fields.
4. Continuum calls the manufacturer API to **look up the distributor** that sold the unit.
5. Portal auto-fills distributor fields.
6. Consumer enters installation details (date, address, type: residential/commercial/multi-family).
7. If a contractor installed the product, Continuum calls the manufacturer API to **search the contractor directory**.
8. Consumer submits. Continuum stores the registration.
9. Continuum calls the manufacturer API to **sync the registration back** — this ensures the manufacturer's warranty database knows the unit is registered, which may activate or extend coverage.

**Warranty Effective Date Logic:** The warranty starts on the **installation date** if documented (via registration). If no install date is recorded, manufacturers typically default to **manufacture date + 90 days**.

---

### 2.2 Warranty Verification

Anyone looks up a serial number to check what's covered.

1. User enters serial number.
2. Continuum retrieves unit details and **warranty coverage tiers** from the manufacturer API.
3. Portal displays: product ID, warranty status broken out by tier (tank, parts, labor — each with their own expiration), installation/registration status, homeowner info.
4. User can view **replacement parts** (BOM with images and specs) and **product documentation** (manuals, diagrams, spec sheets) — all fetched from the manufacturer's middleware.

**Coverage is tiered.** A single "warranty expiration" doesn't exist. Tank might be covered 12 years, parts 12 years, labor only 1 year. Some tiers become pro-rated after an initial full-coverage period.

---

### 2.3 Warranty Claim Processing — The Core Flow

A contractor diagnoses a failure on a consumer's unit and needs to file a warranty claim.

**Phase 1 — Unit Identification & Entitlement (Manufacturer API):**
- Contractor enters serial number → Continuum validates it
- Continuum checks entitlement → returns coverage tiers, labor rates, homeowner info, claim submission deadline

**Phase 2 — Failure Details & Parts Selection (Manufacturer API):**
- Continuum retrieves available **claim types** (Continuum-managed)
- Continuum retrieves the **BOM** so the contractor can identify which part(s) failed
- For each failed part, Continuum retrieves **warranty codes** (hierarchical failure classification tree)
- Contractor selects replacement parts from the **replacements** endpoint

**Phase 3 — Claim Submission (Continuum-managed):**
- Continuum **pre-validates** the claim (dry run checking entitlement, parts, codes, required fields)
- If valid, Continuum **creates the claim** — returns claim number, financial details, whether attachments or part returns are required

**Phase 4 — Attachments (Continuum-managed):**
- If required, contractor **uploads photos/documents** (rating plate, failed part, proof of purchase)

**Phase 5 — Tracking & Resolution (Continuum-managed):**
- Continuum tracks claim status through its lifecycle: `PENDING` → `APPROVED` → `PAID` (or `DENIED`, `NEED_CORRECTIONS`, etc.)
- If corrections needed, contractor updates the claim
- If part return required, Continuum provides **return instructions** (shipping address, label, deadline) and tracks **return confirmation**

**Phase 6 — Closure:**
- Claim reaches terminal state (`PAID`, `DENIED`, `CANCELLED`)

---

### 2.4 Media Center

Consumers and contractors access product documentation — manuals, spec sheets, wiring diagrams, part images, and more.

1. User searches by model number or arrives via a part listing.
2. Continuum retrieves all media (documents + images) from the manufacturer's unified **media endpoint**.
3. Portal displays results organized by category with download links and preview images.
4. Manufacturers can also **upload media manually** for content not available via their internal systems.

---

### 2.5 Historical Data Import

One-time (then incremental) pull of existing registration data from the manufacturer's legacy systems.

1. Continuum calls the manufacturer's **import endpoint** with pagination.
2. Processes pages until complete. Supports incremental sync via `since` parameter.
3. Continuum now has the full registration history from day one.

---

## 3. Complete API Surface

### 3.1 Authentication

Both sides use OAuth2 client credentials.

| Endpoint | Method | Owner | Purpose |
|----------|--------|-------|---------|
| `/v1/oauth2/token` | POST | Manufacturer | Exchange credentials for bearer token |

---

### 3.2 Manufacturer Endpoints (23 total)

These are the endpoints the manufacturer builds and hosts. Continuum calls them.

**Unit & Product Data:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/health` | GET | Health check |
| `/v1/validate-unit` | GET | Validate serial, return model(s) |
| `/v1/units/{serial_number}` | GET | Full unit record (product, sales, manufacturing, warranty, installation, registration) |
| `/v1/entitlement-information` | GET | Detailed entitlement: coverage tiers, labor rates, homeowner, claim deadline |
| `/v1/units/{serial_number}/warranty-coverages` | GET | Coverage tiers only (lightweight) |
| `/v1/units/{serial_number}/parts` | GET | BOM with specs, images, cross-references |
| `/v1/units/{serial_number}/distributor` | GET | Distributor that sold the unit |

**Parts & BOM:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/bill-of-materials` | GET | Simplified BOM for claims parts-selection |
| `/v1/parts/cross-reference` | GET | Part supersession chains and brand cross-references |
| `/v1/products/warranty-codes` | GET | Hierarchical failure classification codes |
| `/v1/products/replacements` | GET | Replacement options (exact, superseded, compatible, aftermarket) |

**Media:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/media` | GET | Documents, images, manuals by model or part number |
| `/v1/media/{id}` | GET | Specific media item by ID |
| `/v1/media` | POST | Manual media upload |

**Reference Data:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/contractors` | GET | Contractor directory search |
| `/v1/locations` | GET | Warranty service locations |
| `/v1/corporations` | GET | Authorized distributors list |
| `/v1/corporations/details` | GET | Distributor details |
| `/v1/corporations/customers` | GET | Customers under a distributor |
| `/v1/labor-rates` | GET | Labor/travel reimbursement rate schedule |

**Registration & Import:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/registrations/sync` | POST | Push registration from Continuum to manufacturer |
| `/v1/import/registrations` | GET | Bulk historical registration export (paginated) |

---

### 3.3 Continuum-Managed Endpoints (11 total)

These are hosted by Continuum. The manufacturer does **not** build these. They exist for claims lifecycle management and are documented here for complete process understanding.

**Claims Lifecycle:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/claim-types` | GET | Available claim types (Parts Only, Full Unit, Labor, Parts+Labor) |
| `/v1/claim/validate` | POST | Pre-validate a claim before submission (dry run) |
| `/v1/claim` | POST | Create a new warranty claim |
| `/v1/claim` | PATCH | Update an existing claim (corrections) |
| `/v1/claim` | DELETE | Cancel a warranty claim |
| `/v1/claim/status` | GET | Check claim status and history |

**Attachments:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/claim/upload-attachment` | POST | Upload supporting photos/documents |

**Returns:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/claim/{claimNumber}/return-instructions` | GET | Return shipping details, labels, and deadline |
| `/v1/claim/{claimNumber}/return-confirmation` | POST | Confirm receipt of returned part |

**History:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/units/{serial_number}/claim-history` | GET | Past claims filed against a serial number |

**Notifications (Planned):**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| Webhook subscription | POST | Register callback URL for real-time claim status updates |

---

### 3.4 Claim Statuses

These are the states a claim moves through within Continuum:

| Status | Description |
|--------|-------------|
| `PENDING` | Submitted, awaiting review |
| `APPROVED` | Approved for payment |
| `DENIED` | Denied |
| `NEED_CORRECTIONS` | Corrections required from contractor |
| `NEED_SERIAL_NUMBER` | Part serial number needed |
| `RETURN_PRODUCT` | Failed part requested for return |
| `PRODUCT_COLLECTED` | Returned part received |
| `READY_FOR_PICKUP` | Replacement ready for pickup |
| `PICKED_UP` | Replacement picked up |
| `PAID` | Credit issued |
| `CANCELLED` | Cancelled |
| `ERROR` | System error |

---

## 4. Key Data Concepts

### 4.1 Coverage Tiers

Every unit has multiple coverage tiers, each with its own status and expiration:

```json
"coverages": [
  { "coverageType": "tank", "status": "active", "expirationDate": "2035-06-15", "isProRated": true },
  { "coverageType": "parts", "status": "active", "expirationDate": "2035-06-15" },
  { "coverageType": "labor", "status": "expired", "expirationDate": "2024-06-15" }
]
```

Pro-rated tiers include a schedule showing how reimbursement decreases over time.

### 4.2 Part Cross-References

Parts are identified by their manufacturer part number (MPN) with cross-references for supersession and brand equivalents:

| Type | Example |
|------|---------|
| `supersedes` | SP20042 supersedes SP20041 |
| `brand_xref` | Rheem SP20042 = Ruud SP20042-RU |
| `oem_xref` | Rheem SP20042 = Cam Mfg A-29-075-AL |
| `upc` | 020352330426 |

### 4.3 The `extendedAttributes` Pattern

Every entity supports a flexible `extendedAttributes[]` array for data that doesn't fit the standard schema — product specs, labor rates, ERP codes, industry-specific fields. The `section` field enables UI grouping.

```json
{ "name": "CapacityGallons", "value": "50", "section": "Capacity" }
```

This keeps the API ERP-agnostic and industry-agnostic. Water heater specs are different from HVAC specs — `extendedAttributes` handles both without schema changes.

### 4.4 Unified Media

All documents, images, manuals, and diagrams are served through a single `/v1/media` endpoint. Query by model number for product documentation, by part number for part images, or by ID for a specific item. The manufacturer's middleware sources media from wherever it lives internally.

---

## 5. Identified Gaps & Open Items

### Resolved

| # | Gap | Resolution |
|---|-----|-----------|
| 1 | Warranty coverage tiers | `coverages[]` array on entitlement response |
| 2 | Installation date missing | `InstallationInfo` on `UnitDetails` |
| 3 | Registration write-back | `POST /v1/registrations/sync` |
| 4 | Claim history per serial | `GET /v1/units/{sn}/claim-history` (Continuum-managed) |
| 5 | Labor/travel rate tables | `GET /v1/labor-rates` + inline on entitlement response |
| 6 | Parts return workflow | Return-instructions + return-confirmation endpoints |
| 7 | Product brand field | `brand` on `ProductInfo` |
| 8 | Pro-rated coverage | `ProRatedSchedule` on `WarrantyCoverage` |
| 9 | Claim submission deadline | `claimSubmissionDeadlineDays` on entitlement |

### Still Open

| # | Gap | Impact | Priority |
|---|-----|--------|----------|
| 1 | **Authorization number** | No pre-approval reference for contractor-to-distributor exchange | Critical |
| 2 | **Contractor certification enforcement** | Non-certified contractors may file claims that get denied | High |
| 3 | **Warranty transferability** | Can't handle ownership changes | Moderate |
| 4 | **Structured attachment validation** | Manufacturer can't auto-validate document completeness | Moderate |
| 5 | **Webhooks** | Must poll for claim status; delayed updates | Moderate |
| 6 | **Replacement unit warranty linking** | Can't link replacement to original claim/coverage | Moderate |
| 7 | **Proof of purchase rules** | No structured field for when required or what qualifies | Moderate |

---

## 6. Next Steps

1. **Resolve the authorization number gap** — This is the last critical blocker. Needs design discussion.
2. **Review Continuum endpoint schemas** with the engineering team — validate claim creation request/response structure against what's actually built.
3. **Build OpenAPI spec** from the detailed specs in `/specs/endpoints.md` and `/specs/data-types.md`.
4. **Publish the Mintlify documentation portal** — already built and ready to deploy.
5. **Begin manufacturer onboarding** — use the implementation guide as the kickoff document.

---

*This is a living document. Detailed endpoint specs with full request/response schemas are in `/specs/endpoints.md`. Data type definitions are in `/specs/data-types.md`. The published documentation portal is in the Mintlify repo root.*
