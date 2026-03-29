# Continuum Manufacturer Warranty Hub — Complete API Specification

**Version:** 1.2-draft  
**Date:** March 29, 2026  
**Status:** Draft  
**Authentication:** OAuth2 Client Credentials  

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Key Concepts](#3-key-concepts)
4. [Process Flows](#4-process-flows)
5. [Authentication](#5-authentication)
6. [Shared Data Types](#6-shared-data-types)
7. [Manufacturer Endpoints (23)](#7-manufacturer-endpoints)
8. [Continuum-Managed Endpoints (11)](#8-continuum-managed-endpoints)
9. [Error Handling](#9-error-handling)
10. [Gap Analysis & Open Items](#10-gap-analysis--open-items)

---

## 1. Overview

### What This Is

Continuum is a warranty and returns automation platform. For the Manufacturer Warranty Hub, Continuum operates as the **consumer-facing warranty portal** on behalf of the manufacturer. Consumers, contractors, and distributors interact with Continuum's UI. Behind the scenes, Continuum calls into the manufacturer's middleware API to retrieve data from and push updates to their systems of record.

This document is the **complete API specification** covering both sides of the integration:

- **Manufacturer endpoints (Section 7)** — The 23 endpoints the manufacturer builds and hosts. Continuum calls these.
- **Continuum-managed endpoints (Section 8)** — The 11 endpoints Continuum owns internally for claims processing, attachments, and status tracking.

### Why This Matters

Warranty processing in manufacturing is painfully manual. A typical claim involves a homeowner calling a toll-free number, a phone agent typing a serial number into a green-screen terminal, manually verifying coverage, issuing an authorization number over the phone, and eventually processing a paper claim form weeks later. Continuum automates this entire lifecycle.

### Key Actors

| Actor | Role | System |
|-------|------|--------|
| **Consumer / Homeowner** | Owns the product. Registers products, checks warranty, initiates service. | Continuum web portal |
| **Contractor** | Diagnoses failures, performs repairs, files warranty claims. | Continuum contractor portal |
| **Distributor** | Sells products, processes warranty exchanges at the counter. | Continuum UI + own systems |
| **Manufacturer** | Makes the product. Owns warranty terms, parts, claims processing. | ERP, warranty DB (behind middleware) |
| **Continuum** | Orchestrates the process. Provides UI, automates data flows. | Warranty platform |

---

## 2. Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│   Consumer /     │     │                  │     │   Manufacturer's     │     │   Manufacturer's │
│   Contractor /   │────▶│   Continuum      │────▶│   Middleware API      │────▶│   Systems of     │
│   Distributor    │◀────│   Platform       │◀────│   (Manufacturer      │◀────│   Record         │
│                  │     │                  │     │    builds this)       │     │                  │
│   Web Browser    │     │  Warranty Hub    │     │  JSON REST API       │     │  ERP / Warranty  │
└─────────────────┘     └──────────────────┘     └──────────────────────┘     │  DB / Legacy     │
                                                                              └──────────────────┘
```

**Communication:** All traffic is REST over HTTPS, authenticated via OAuth2 bearer tokens. Continuum initiates all calls to the manufacturer's middleware. The manufacturer's middleware never calls Continuum directly (except for optional webhook notifications).

**Endpoint ownership:**
- **Manufacturer hosts:** Unit lookups, entitlement checks, parts/BOM, media, reference data, registration sync, historical import (Section 7)
- **Continuum hosts:** Claims CRUD, claim status, attachments, claim types, claim history, return workflow (Section 8)

---

## 3. Key Concepts

### Warranty Effective Date

The warranty start date is determined by:
1. **Installation date** — if the product has been registered and the install date is documented.
2. **Manufacture date + 90 days** — fallback if no installation date is recorded.

### Coverage Tiers

Manufacturers have multiple coverage tiers with different expiration dates:

| Tier | Typical Duration | Example |
|------|-----------------|---------|
| Tank / Heat Exchanger | 6–15 years | Full replacement years 1–6, pro-rated years 7–12 |
| Component Parts | 5–12 years | Free parts for the term |
| Labor | 1–3 years | Reimbursement for contractor labor |
| Component-specific | Varies | Compressor = 10 years, LIFEGUARD elements = 3 years |

Coverage may differ for residential vs. commercial installations, and may be pro-rated after an initial full-coverage period.

### Part Cross-References

A single part is often known by multiple identifiers:
- **Supersession** — new part number replaces an older one
- **Brand cross-reference** — same part under different brand names within the manufacturer's portfolio
- **OEM cross-reference** — the component supplier's part number
- **UPC / GTIN** — universal barcode

### The `extendedAttributes` Pattern

Every entity supports an `extendedAttributes[]` array of name-value pairs for data that doesn't fit the standard schema. This is the primary mechanism for product specifications, industry-specific data, labor rate structures, and ERP-specific fields.

```json
{ "name": "CapacityGallons", "value": "50", "section": "Capacity" }
```

The `section` field enables UI grouping. Continuum renders these as grouped attribute tables.

### Unified Media Model

All media (documents, images, manuals, diagrams) is accessed through a single `/v1/media` endpoint, queryable by model number, part number, or media ID.

---

## 4. Process Flows

### 4.1 Product Registration

**Sequence:** Consumer enters serial → Continuum validates → auto-fills product/distributor → consumer completes form → Continuum syncs to manufacturer

| Step | API Call | Owner |
|------|----------|-------|
| 1 | `GET /v1/validate-unit?SerialNumber={sn}` | Manufacturer |
| 2 | `GET /v1/units/{serial}` | Manufacturer |
| 3 | `GET /v1/units/{serial}/distributor` | Manufacturer |
| 4 | `GET /v1/contractors?zip_code={zip}` | Manufacturer |
| 5 | `POST /v1/registrations/sync` | Manufacturer |

### 4.2 Warranty Verification

**Sequence:** User enters serial → Continuum retrieves unit + coverage + parts + media

| Step | API Call | Owner |
|------|----------|-------|
| 1 | `GET /v1/units/{serial}` | Manufacturer |
| 2 | `GET /v1/units/{serial}/warranty-coverages` | Manufacturer |
| 3 | `GET /v1/units/{serial}/parts` | Manufacturer |
| 4 | `GET /v1/media?model_number={model}` | Manufacturer |

### 4.3 Claims Processing

**Phase 1 — Unit identification & entitlement (Manufacturer endpoints):**

| Step | API Call | Owner |
|------|----------|-------|
| 1 | `GET /v1/validate-unit?SerialNumber={sn}` | Manufacturer |
| 2 | `GET /v1/entitlement-information?SerialNumber={sn}` | Manufacturer |

**Phase 2 — Failure details & parts selection (Manufacturer endpoints):**

| Step | API Call | Owner |
|------|----------|-------|
| 3 | `GET /v1/claim-types` | Continuum |
| 4 | `GET /v1/bill-of-materials?SerialNumber={sn}` | Manufacturer |
| 5 | `GET /v1/products/warranty-codes?PartNumber={pn}` | Manufacturer |
| 6 | `GET /v1/products/replacements?PartNumber={pn}` | Manufacturer |

**Phase 3 — Claim submission (Continuum endpoints):**

| Step | API Call | Owner |
|------|----------|-------|
| 7 | `POST /v1/claim/validate` | Continuum |
| 8 | `POST /v1/claim` | Continuum |

**Phase 4 — Attachments (Continuum endpoints):**

| Step | API Call | Owner |
|------|----------|-------|
| 9 | `POST /v1/claim/upload-attachment` | Continuum |

**Phase 5 — Tracking & resolution (Continuum endpoints):**

| Step | API Call | Owner |
|------|----------|-------|
| 10 | `GET /v1/claim/status?ClaimNumber={cn}` | Continuum |
| 11 | `PATCH /v1/claim` (if corrections needed) | Continuum |
| 12 | `GET /v1/claim/{claimNumber}/return-instructions` | Continuum |
| 13 | `POST /v1/claim/{claimNumber}/return-confirmation` | Continuum |

**Phase 6 — Closure:** Claim reaches terminal state (`PAID`, `DENIED`, `CANCELLED`).

### 4.4 Media Center

| Step | API Call | Owner |
|------|----------|-------|
| 1 | `GET /v1/media?model_number={model}` | Manufacturer |
| 2 | `GET /v1/media?part_number={part}` | Manufacturer |
| 3 | `GET /v1/media/{id}` | Manufacturer |

### 4.5 Historical Import

| Step | API Call | Owner |
|------|----------|-------|
| 1 | `GET /v1/import/registrations?page=1&page_size=100` | Manufacturer |
| 2 | Repeat with incremental `since` parameter | Manufacturer |

---

## 5. Authentication

### `POST /v1/oauth2/token`

**Owner:** Manufacturer  
**Content-Type:** `application/json`

**Request:**
```json
{
  "grant_type": "client_credentials",
  "client_id": "continuum-warranty-hub",
  "client_secret": "••••••••••••••••",
  "scope": "warranty:read warranty:write"
}
```

**Response `200 OK`:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "warranty:read warranty:write"
}
```

All subsequent calls include: `Authorization: Bearer {access_token}`

---

## 6. Shared Data Types

### Address

```json
{
  "address1": "string | null",
  "address2": "string | null",
  "address3": "string | null",
  "city": "string | null",
  "state": "string | null",
  "stateCode": "string | null",
  "zip": "string | null",
  "country": "string | null",
  "countryCode": "string | null"
}
```

### DataAttribute

```json
{
  "name": "string",         // Required
  "value": "string | null",
  "section": "string | null" // UI grouping key
}
```

### HomeownerInfo

```json
{
  "id": "string | null",
  "businessName": "string | null",
  "firstName": "string | null",
  "lastName": "string | null",
  "email": "string | null",
  "phone": "string | null",
  "locationId": "string | null",
  "address": { /* Address */ }
}
```

### ContractorInfo

```json
{
  "id": "string | null",
  "firstName": "string | null",
  "lastName": "string | null",
  "companyName": "string | null",
  "email": "string | null",
  "phone": "string | null",
  "licenseNumber": "string | null",
  "isCertified": "boolean | null",
  "certifications": ["string"],
  "locationId": "string | null",
  "address": { /* Address */ }
}
```

### DistributorInfo

```json
{
  "distributorId": "string | null",
  "name": "string | null",
  "accountNumber": "string | null",
  "purchaseDate": "date | null",
  "address": { /* Address */ },
  "phone": "string | null",
  "email": "string | null"
}
```

### ProductInfo

```json
{
  "baseModel": "string | null",
  "modelDescription": "string | null",
  "sku": "string | null",
  "materialNumber": "string | null",
  "materialDescription": "string | null",
  "brand": "string | null",
  "productLine": "string | null",
  "seriesName": "string | null",
  "productCategory": "string | null",
  "productType": "string | null",
  "fuelType": "string | null",
  "extendedAttributes": [ /* DataAttribute[] */ ]
}
```

### ManufacturingInfo

```json
{
  "plantCode": "string | null",
  "plantName": "string | null",
  "manufactureDate": "date | null",
  "lotNumber": "string | null",
  "countryOfOrigin": "string | null"
}
```

### SalesInfo

```json
{
  "salesOrg": "string | null",
  "salesOrderNumber": "string | null",
  "originalPoNumber": "string | null",
  "shippingPlant": "string | null",
  "customerId": "string | null",
  "customerName": "string | null",
  "shipDate": "date | null",
  "invoiceNumber": "string | null",
  "invoiceDate": "date | null"
}
```

### InstallationInfo

```json
{
  "installDate": "date | null",
  "installationType": "string | null",
  "installedBy": { /* ContractorInfo */ },
  "installationAddress": { /* Address */ },
  "isProperlyDocumented": "boolean | null"
}
```

`installDate` is the warranty effective date if documented. Falls back to `manufactureDate + 90 days` if null.

### RegistrationInfo

```json
{
  "isRegistered": "boolean",       // Required
  "registrationDate": "datetime | null",
  "registrationSource": "string | null",
  "registeredOwner": { /* HomeownerInfo */ },
  "affectsWarrantyTerms": "boolean | null",
  "extendedCoverageDays": "integer | null"
}
```

### WarrantySummary

```json
{
  "overallStatus": "string",       // Required: "produced", "active", "partial", "expired", "voided", "pending"
  "effectiveDate": "date | null",
  "isTransferable": "boolean | null",
  "currentOwnerIsOriginal": "boolean | null"
}
```

### WarrantyCoverage

```json
{
  "coverageType": "string",        // Required: "tank", "heat_exchanger", "parts", "labor", "component_specific"
  "coverageLabel": "string | null",
  "status": "string",              // Required: "active", "expired", "voided", "pending", "pro_rated"
  "effectiveDate": "date | null",
  "expirationDate": "date | null",
  "coverageTermMonths": "integer | null",
  "isProRated": "boolean | null",
  "proRatedSchedule": { /* ProRatedSchedule | null */ },
  "coveredComponents": ["string"],
  "exclusions": ["string"],
  "requiresRegistration": "boolean | null",
  "extendedAttributes": [ /* DataAttribute[] */ ]
}
```

### ProRatedSchedule

```json
{
  "fullCoverageMonths": 60,
  "tiers": [
    { "fromMonth": 1, "toMonth": 60, "reimbursementPercent": 100.0, "description": "Full replacement" },
    { "fromMonth": 61, "toMonth": 84, "reimbursementPercent": 75.0, "description": "75% credit" },
    { "fromMonth": 85, "toMonth": 120, "reimbursementPercent": 50.0, "description": "50% credit" }
  ]
}
```

### LaborRateSchedule

```json
{
  "effectiveDate": "date | null",
  "expirationDate": "date | null",
  "extendedAttributes": [ /* DataAttribute[] — all rate data */ ]
}
```

Example attributes: `LaborRate`, `LaborRateUOM`, `MaxLaborHours`, `TravelRatePerMile`, `MaxTravelMiles`, `RefrigerantAllowance`.

### PartCrossReference

```json
{
  "referenceType": "string",  // Required: "supersedes", "superseded_by", "brand_xref", "oem_xref", "upc", "equivalent"
  "partNumber": "string",     // Required
  "brand": "string | null",
  "description": "string | null"
}
```

### BOMPart

```json
{
  "partNumber": "string",             // Required (MPN)
  "description": "string | null",
  "crossReferences": [ /* PartCrossReference[] */ ],
  "quantity": "number | null",
  "unitOfMeasure": "string | null",
  "partType": "string | null",
  "partSubType": "string | null",
  "brand": "string | null",
  "isServiceable": "boolean | null",
  "isWarrantyCovered": "boolean | null",
  "warrantyExpirationDate": "date | null",
  "inStock": "boolean | null",
  "images": [ /* MediaItem[] */ ],
  "notes": "string | null",
  "replacementsAvailable": "integer | null",
  "extendedAttributes": [ /* DataAttribute[] — part specifications */ ]
}
```

### ReplacementPart

```json
{
  "partNumber": "string",        // Required
  "description": "string | null",
  "crossReferences": [ /* PartCrossReference[] */ ],
  "brand": "string | null",
  "isOEM": "boolean",            // Required
  "replacementType": "string | null",  // "exact", "superseded", "compatible", "aftermarket", "universal"
  "inStock": "boolean | null",
  "leadTimeDays": "integer | null",
  "listPrice": "number | null",
  "warrantyPrice": "number | null",
  "score": "integer | null",
  "scoreDetail": "string | null",
  "images": [ /* MediaItem[] */ ],
  "extendedAttributes": [ /* DataAttribute[] */ ]
}
```

### WarrantyCode

Hierarchical tree: general → specific → detail.

```json
{
  "id": "string",                     // Required
  "description": "string | null",
  "fullPath": "string | null",
  "idPath": "string | null",
  "requiresSerialNumber": "boolean | null",
  "requiresAttachment": "boolean | null",
  "children": [ /* WarrantyCode[] */ ]  // Empty = leaf node (selectable)
}
```

### MediaItem

```json
{
  "id": "string",              // Required
  "title": "string | null",
  "filename": "string | null",
  "mediaType": "string",       // Required: "document", "image", "video", "diagram"
  "category": "string | null",
  "url": "string",             // Required
  "previewUrl": "string | null",
  "mimeType": "string | null",
  "fileSizeBytes": "integer | null",
  "pageCount": "integer | null",
  "width": "integer | null",
  "height": "integer | null",
  "lastUpdated": "date | null",
  "language": "string | null",
  "associatedModels": ["string"],
  "associatedPartNumbers": ["string"],
  "extendedAttributes": [ /* DataAttribute[] */ ]
}
```

**Document categories:** `installation_guide`, `service_manual`, `parts_list`, `warranty_information`, `spec_sheet`, `wiring_diagram`, `troubleshooting_guide`, `safety_instructions`, `owners_manual`, `energy_guide`, `dimensional_drawing`, `product_data`, `service_bulletin`, `iom`

**Image categories:** `product_photo`, `part_photo`, `installation_photo`, `diagram`, `exploded_view`, `rating_plate`

### UnitDetails (Composite)

```json
{
  "serialNumber": "string",
  "product": { /* ProductInfo */ },
  "manufacturing": { /* ManufacturingInfo */ },
  "sales": { /* SalesInfo */ },
  "warranty": { /* WarrantySummary */ },
  "installation": { /* InstallationInfo */ },
  "registration": { /* RegistrationInfo */ },
  "extendedAttributes": [ /* DataAttribute[] */ ]
}
```

### PaginatedResponse

```json
{
  "pagination": {
    "page": "integer",
    "pageSize": "integer",
    "totalRecords": "integer",
    "totalPages": "integer"
  }
}
```

---

## 7. Manufacturer Endpoints

These 23 endpoints are hosted by the manufacturer. Continuum calls them.

**Base URL:** `https://{manufacturer-host}/v1`

---

### 7.1 `GET /v1/health`

Health check. Continuum polls periodically.

**Response `200`:**
```json
{ "status": "Healthy", "timestamp": "2026-03-29T14:30:00Z", "version": "1.0" }
```

---

### 7.2 `GET /v1/validate-unit`

Lightweight serial number validation. First call in any workflow.

**Query:** `SerialNumber` (required), `ModelNumber` (optional)

**Response `200`:**
```json
{
  "serialNumber": "ABC123456789",
  "modelNumbers": [
    { "modelNumber": "EN6-50-DORT", "modelDescription": "50gal Tall EL 4.5kW 240V", "installDate": "2023-06-15T00:00:00Z" }
  ]
}
```

---

### 7.3 `GET /v1/units/{serial_number}`

Full unit record. See `UnitDetails` data type for the complete response schema.

**Path:** `serial_number` (required)

**Response `200`:** Returns `UnitDetails` with product info (including `extendedAttributes` for technical specs), manufacturing, sales, warranty summary, installation, and registration data.

*See Section 6 for the full UnitDetails schema. See the specs/endpoints.md file for the complete JSON example.*

---

### 7.4 `GET /v1/entitlement-information`

Primary entitlement check before filing a claim. Returns per-tier coverage, labor rates, homeowner info, and purchaser details.

**Query:** `SerialNumber` (required), `ModelNumber`, `RegisteredOwnerLastName`, `StateCode`, `CustomerId`, `InstallationType`

**Response `200`:**
```json
{
  "serialNumberIsValid": true,
  "warrantyIsValid": true,
  "serialNo": "ABC123456789",
  "modelId": "EN6-50-DORT",
  "brand": "Rheem",
  "installationType": "Residential",
  "installDate": "2023-06-15T00:00:00Z",
  "registrationDate": "2023-06-20T00:00:00Z",
  "isSerialNumberRegistered": true,
  "customerIdIsValid": true,
  "coverages": [ /* WarrantyCoverage[] — per-tier breakdown */ ],
  "laborRates": { /* LaborRateSchedule */ },
  "claimSubmissionDeadlineDays": 30,
  "homeowner": { /* HomeownerInfo */ },
  "purchaserDetails": {
    "distributorName": "Premier Plumbing Supply",
    "address": { /* Address */ }
  },
  "extendedAttributes": []
}
```

---

### 7.5 `GET /v1/units/{serial_number}/warranty-coverages`

Coverage tiers only. Lightweight alternative to full entitlement.

**Path:** `serial_number` (required)

**Response `200`:**
```json
{
  "serialNumber": "ABC123456789",
  "overallStatus": "partial",
  "effectiveDate": "2023-06-15",
  "coverages": [ /* WarrantyCoverage[] */ ],
  "extendedAttributes": []
}
```

---

### 7.6 `GET /v1/units/{serial_number}/parts`

BOM with specs, images, and cross-references.

**Path:** `serial_number` (required)  
**Query:** `include_images` (default true), `in_stock_only`, `serviceable_only` (default true)

**Response `200`:**
```json
{
  "serialNumber": "ABC123456789",
  "baseModel": "EN6-50-DORT",
  "parts": [ /* BOMPart[] */ ],
  "totalParts": 8
}
```

---

### 7.7 `GET /v1/units/{serial_number}/distributor`

Distributor that originally sold the unit.

**Path:** `serial_number` (required)

**Response `200`:** Returns `DistributorInfo`.

---

### 7.8 `GET /v1/bill-of-materials`

Simplified BOM for claims parts-selection. Lighter than `/units/{sn}/parts`.

**Query:** `SerialNumber` (required), `ModelNumber`

**Response `200`:**
```json
{
  "serialNumber": "ABC123456789",
  "modelNumber": "EN6-50-DORT",
  "parts": [
    { "partNumber": "SP20042", "description": "K,ANODE,29\"...", "partType": "anode", "crossReferences": [...] }
  ]
}
```

---

### 7.9 `GET /v1/parts/cross-reference`

Part supersession chains and brand cross-references.

**Query:** `partNumber` (required)

**Response `200`:**
```json
{
  "partNumber": "SP20042",
  "description": "K,ANODE,29\"...",
  "brand": "Rheem",
  "crossReferences": [
    { "referenceType": "supersedes", "partNumber": "SP20041", "description": "Previous version" },
    { "referenceType": "brand_xref", "partNumber": "SP20042-RU", "brand": "Ruud" },
    { "referenceType": "superseded_by", "partNumber": "SP20042-V2", "description": "Newer version" },
    { "referenceType": "oem_xref", "partNumber": "A-29-075-AL", "brand": "Cam Manufacturing" },
    { "referenceType": "upc", "partNumber": "020352330426" }
  ]
}
```

---

### 7.10 `GET /v1/products/warranty-codes`

Hierarchical warranty disposition codes for failure classification.

**Query:** `SerialNumber`, `ModelNumber`, `PartNumber`

**Response `200`:**
```json
{
  "serialNumberRequired": false,
  "reasonCodes": [ /* WarrantyCode[] — recursive tree */ ]
}
```

---

### 7.11 `GET /v1/products/replacements`

Replacement options for a failed part.

**Query:** `PartNumber` (required), `IncludeNonOEM` (default false)

**Response `200`:**
```json
{
  "replacementParts": [ /* ReplacementPart[] */ ]
}
```

---

### 7.12 `GET /v1/media`

Unified media retrieval by model or part.

**Query:** `model_number` or `part_number` (at least one required), `mediaType`, `category`

**Response `200`:**
```json
{
  "media": [ /* MediaItem[] */ ],
  "total": 3
}
```

---

### 7.13 `GET /v1/media/{id}`

Specific media item by ID.

**Path:** `id` (required)

**Response `200`:** Returns `MediaItem`.

---

### 7.14 `POST /v1/media`

Manual media upload. `multipart/form-data`.

**Fields:** `file` (binary, required), `title` (required), `mediaType` (required), `category` (required), `model_number`, `part_number`

**Response `201`:**
```json
{ "id": "media-new-001", "url": "https://...", "uploadedAt": "2024-07-20T10:00:00Z" }
```

---

### 7.15 `GET /v1/contractors`

Contractor directory search.

**Query:** `search`, `zip_code`, `certified_only`, `page`, `page_size`

**Response `200`:**
```json
{
  "contractors": [ /* ContractorInfo[] */ ],
  "pagination": { /* PaginatedResponse */ }
}
```

---

### 7.16 `GET /v1/locations`

Warranty service locations.

**Response `200`:**
```json
{
  "locations": [
    { "locationId": "LOC-001", "locationName": "Nashville DC", "locationType": "distribution_center", "address": {...}, "isActive": true }
  ]
}
```

---

### 7.17 `GET /v1/corporations`

Authorized distributors.

**Response `200`:**
```json
{
  "corporations": [
    { "corporationId": "CORP-001", "corporationName": "Premier Plumbing Supply", "extendedAttributes": [] }
  ]
}
```

---

### 7.18 `GET /v1/corporations/details`

Distributor details.

**Query:** `CorporationId` (required)

**Response `200`:** Returns corporation with `extendedAttributes` (region, tier, etc.).

---

### 7.19 `GET /v1/corporations/customers`

Customers under a distributor.

**Query:** `CorporationId` (required)

**Response `200`:**
```json
{
  "customers": [
    { "companyId": "C-827451", "companyName": "Smith Plumbing LLC", "extendedAttributes": [] }
  ]
}
```

---

### 7.20 `GET /v1/labor-rates`

Labor and travel reimbursement rates. All rate data in `extendedAttributes`.

**Query:** `StateCode`, `ZipCode`, `InstallationType`, `ClaimType`

**Response `200`:**
```json
{
  "rates": [ /* LaborRateSchedule[] */ ]
}
```

---

### 7.21 `POST /v1/registrations/sync`

Push registration from Continuum to manufacturer.

**Request:**
```json
{
  "serialNumber": "ABC123456789",
  "modelNumber": "EN6-50-DORT",
  "registrationDate": "2024-01-15T14:30:00Z",
  "installDate": "2024-01-10",
  "installationType": "Residential",
  "homeowner": { /* HomeownerInfo */ },
  "contractor": { /* ContractorInfo */ },
  "distributor": { /* DistributorInfo */ },
  "purchaseDate": "2024-01-05"
}
```

**Response `200`:**
```json
{
  "synced": true,
  "manufacturerRegistrationId": "REG-MFR-2024-00456",
  "warrantyActivated": true,
  "coverageAdjustments": [
    { "coverageType": "tank", "previousExpirationDate": null, "newExpirationDate": "2036-01-10", "reason": "Registration activated full warranty" }
  ],
  "message": "Registration synced successfully."
}
```

**Response `409`:** Already registered.

---

### 7.22 `GET /v1/import/registrations`

Bulk historical registration export. Paginated.

**Query:** `since` (ISO 8601), `page` (default 1), `page_size` (default 100, max 1000)

**Response `200`:**
```json
{
  "registrations": [
    {
      "serialNumber": "ABC123456789",
      "modelNumber": "EN6-50-DORT",
      "brand": "Rheem",
      "registrationDate": "2022-06-15T00:00:00Z",
      "installDate": "2022-06-10",
      "installationType": "Residential",
      "distributor": { /* DistributorInfo */ },
      "homeowner": { /* HomeownerInfo */ },
      "warranty": { /* WarrantySummary */ },
      "coverages": [ /* WarrantyCoverage[] */ ]
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "totalRecords": 45230, "totalPages": 453 }
}
```

---

### 7.23 Endpoint Summary — Manufacturer

| # | Method | Endpoint | Purpose |
|---|--------|----------|---------|
| 1 | POST | `/v1/oauth2/token` | Authentication |
| 2 | GET | `/v1/health` | Health check |
| 3 | GET | `/v1/validate-unit` | Validate serial, return model(s) |
| 4 | GET | `/v1/units/{serial_number}` | Full unit record |
| 5 | GET | `/v1/entitlement-information` | Warranty entitlement + coverages |
| 6 | GET | `/v1/units/{serial_number}/warranty-coverages` | Coverage tiers only |
| 7 | GET | `/v1/units/{serial_number}/parts` | BOM with specs + images |
| 8 | GET | `/v1/units/{serial_number}/distributor` | Distributor lookup |
| 9 | GET | `/v1/bill-of-materials` | BOM for claims context |
| 10 | GET | `/v1/parts/cross-reference` | Part supersedes + brand cross-refs |
| 11 | GET | `/v1/products/warranty-codes` | Hierarchical warranty codes |
| 12 | GET | `/v1/products/replacements` | Replacement part options |
| 13 | GET | `/v1/media` | Media by model or part |
| 14 | GET | `/v1/media/{id}` | Specific media item by ID |
| 15 | POST | `/v1/media` | Manual media upload |
| 16 | GET | `/v1/contractors` | Contractor directory |
| 17 | GET | `/v1/locations` | Warranty locations |
| 18 | GET | `/v1/corporations` | Authorized distributors |
| 19 | GET | `/v1/corporations/details` | Distributor details |
| 20 | GET | `/v1/corporations/customers` | Customers under distributor |
| 21 | GET | `/v1/labor-rates` | Labor/travel rate schedule |
| 22 | POST | `/v1/registrations/sync` | Push registration to manufacturer |
| 23 | GET | `/v1/import/registrations` | Bulk historical registration export |

---

## 8. Continuum-Managed Endpoints

These 11 endpoints are hosted by Continuum. They are **not** part of the manufacturer's middleware implementation. They are documented here for complete process understanding.

**Base URL:** `https://api.gocontinuum.ai/v1`

---

### 8.1 `GET /v1/claim-types`

Returns available claim types for the claims form.

**Response `200`:**
```json
{
  "claimTypes": [
    { "id": "PARTS_ONLY", "description": "Parts Only — replacement part(s) under warranty" },
    { "id": "FULL_UNIT", "description": "Full Unit Replacement — complete unit swap" },
    { "id": "LABOR_CLAIM", "description": "Labor Claim — reimbursement for labor performed" },
    { "id": "PARTS_AND_LABOR", "description": "Parts and Labor — combined claim" }
  ]
}
```

---

### 8.2 `POST /v1/claim/validate`

Pre-validates a claim before submission (dry run). Checks entitlement, parts coverage, warranty codes, and required fields.

**Request:** Same structure as `POST /v1/claim` (see 8.3).

**Response `200`:**
```json
{
  "isValid": true,
  "warnings": [],
  "errors": [],
  "requiredAttachments": ["rating_plate_photo"],
  "estimatedReimbursement": {
    "parts": 45.99,
    "labor": 170.00,
    "travel": 33.50,
    "total": 249.49
  }
}
```

**Response `422`:**
```json
{
  "isValid": false,
  "errors": [
    { "field": "warrantyCodeId", "message": "Warranty code ELEC/THERM/FAIL-REG requires an attachment" },
    { "field": "laborHours", "message": "Labor hours exceed maximum allowable (2.0) for this claim type" }
  ]
}
```

---

### 8.3 `POST /v1/claim`

Create a new warranty claim.

**Request:**
```json
{
  "serialNumber": "ABC123456789",
  "modelNumber": "EN6-50-DORT",
  "claimType": "PARTS_AND_LABOR",
  "failureDate": "2026-03-25",
  "failureDescription": "Upper thermostat not responding. No hot water.",
  "homeowner": { /* HomeownerInfo */ },
  "contractor": { /* ContractorInfo */ },
  "customerId": "5678901",
  "installationType": "Residential",
  "lineItems": [
    {
      "failedPartNumber": "SP8346",
      "failedPartDescription": "THERMOSTAT,UPPER,SNAP-DISC,170F",
      "warrantyCodeId": "ELEC/THERM/NO-RESP",
      "replacementPartNumber": "SP8346",
      "quantity": 1,
      "notes": "No power to upper thermostat. Confirmed with multimeter."
    }
  ],
  "labor": {
    "hours": 1.5,
    "rate": 85.00,
    "travelMiles": 22,
    "travelRate": 0.67,
    "notes": null
  }
}
```

**Response `201`:**
```json
{
  "claimNumber": "CLM-2026-00789",
  "status": "PENDING",
  "createdAt": "2026-03-29T15:00:00Z",
  "lineItems": [
    {
      "failedPartNumber": "SP8346",
      "replacementPartNumber": "SP8346",
      "unitPrice": 28.50,
      "warrantyPrice": 0.00,
      "requestedForReturn": false
    }
  ],
  "financials": {
    "partsTotal": 0.00,
    "laborTotal": 127.50,
    "travelTotal": 14.74,
    "claimTotal": 142.24
  },
  "attachmentsRequired": true,
  "attachmentPrompt": "Please upload a photo of the failed thermostat."
}
```

---

### 8.4 `PATCH /v1/claim`

Update an existing claim (corrections, additional info).

**Request:**
```json
{
  "claimNumber": "CLM-2026-00789",
  "failureDescription": "Updated: Upper thermostat failed open. Replaced and tested.",
  "labor": {
    "hours": 2.0
  }
}
```

**Response `200`:** Returns updated claim object.

---

### 8.5 `DELETE /v1/claim`

Cancel/delete a warranty claim.

**Query:** `ClaimNumber` (required)

**Response `200`:**
```json
{
  "claimNumber": "CLM-2026-00789",
  "status": "CANCELLED",
  "cancelledAt": "2026-03-29T16:00:00Z"
}
```

---

### 8.6 `GET /v1/claim/status`

Check claim status.

**Query:** `ClaimNumber` (required)

**Response `200`:**
```json
{
  "claimNumber": "CLM-2026-00789",
  "status": "APPROVED",
  "statusDate": "2026-03-30T09:00:00Z",
  "statusHistory": [
    { "status": "PENDING", "date": "2026-03-29T15:00:00Z" },
    { "status": "APPROVED", "date": "2026-03-30T09:00:00Z" }
  ],
  "financials": {
    "partsTotal": 0.00,
    "laborTotal": 127.50,
    "travelTotal": 14.74,
    "claimTotal": 142.24
  },
  "nextAction": null,
  "notes": null
}
```

**Claim statuses:**

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

### 8.7 `POST /v1/claim/upload-attachment`

Upload supporting documents/photos for a claim. `multipart/form-data`.

**Fields:** `claimNumber` (required), `file` (binary, required), `attachmentType` (recommended: `rating_plate_photo`, `proof_of_purchase`, `failed_part_photo`, `installation_photo`, `other`), `description`

**Response `200`:**
```json
{
  "attachmentId": "ATT-001",
  "claimNumber": "CLM-2026-00789",
  "filename": "thermostat_failed.jpg",
  "uploadedAt": "2026-03-29T15:05:00Z"
}
```

---

### 8.8 `GET /v1/claim/{claimNumber}/return-instructions`

Return shipping details when a failed part is requested for return.

**Path:** `claimNumber` (required)

**Response `200`:**
```json
{
  "claimNumber": "CLM-2026-00789",
  "returnRequired": true,
  "returnDeadlineDays": 30,
  "returnDeadlineDate": "2026-04-29",
  "shippingAddress": { /* Address — manufacturer's return center */ },
  "shippingLabel": {
    "url": "https://labels.example.com/return-label-CLM-2026-00789.pdf",
    "carrier": "UPS",
    "trackingNumber": null
  },
  "instructions": "Package the failed thermostat in the replacement part's box. Affix the return label and drop off at any UPS location.",
  "partsToReturn": [
    { "partNumber": "SP8346", "description": "THERMOSTAT,UPPER,SNAP-DISC,170F", "quantity": 1 }
  ]
}
```

---

### 8.9 `POST /v1/claim/{claimNumber}/return-confirmation`

Confirm receipt of returned part.

**Path:** `claimNumber` (required)

**Request:**
```json
{
  "receivedDate": "2026-04-10T00:00:00Z",
  "receivedBy": "Warehouse Nashville",
  "condition": "as_described",
  "notes": null
}
```

**Response `200`:**
```json
{
  "claimNumber": "CLM-2026-00789",
  "returnStatus": "received",
  "claimStatus": "PRODUCT_COLLECTED"
}
```

---

### 8.10 `GET /v1/units/{serial_number}/claim-history`

Past claims filed against a serial number.

**Path:** `serial_number` (required)

**Response `200`:**
```json
{
  "serialNumber": "ABC123456789",
  "claims": [
    {
      "claimNumber": "CLM-2025-00123",
      "claimType": "PARTS_ONLY",
      "status": "PAID",
      "createdAt": "2025-08-15T00:00:00Z",
      "resolvedAt": "2025-08-22T00:00:00Z",
      "failedParts": ["SP20041"],
      "replacementParts": ["SP20042"],
      "claimTotal": 45.99
    }
  ],
  "totalClaims": 1
}
```

---

### 8.11 Webhook Subscription (Future)

**Status:** Planned, not yet implemented.

Continuum will support webhook callbacks for real-time claim status notifications, eliminating the need to poll `GET /v1/claim/status`. Manufacturers would register a callback URL and receive `POST` payloads when claim status changes.

---

### 8.12 Endpoint Summary — Continuum

| # | Method | Endpoint | Purpose |
|---|--------|----------|---------|
| 1 | GET | `/v1/claim-types` | Available claim types |
| 2 | POST | `/v1/claim/validate` | Pre-validate a claim |
| 3 | POST | `/v1/claim` | Create warranty claim |
| 4 | PATCH | `/v1/claim` | Update warranty claim |
| 5 | DELETE | `/v1/claim` | Cancel warranty claim |
| 6 | GET | `/v1/claim/status` | Check claim status |
| 7 | POST | `/v1/claim/upload-attachment` | Upload claim attachments |
| 8 | GET | `/v1/claim/{claimNumber}/return-instructions` | Return shipping details |
| 9 | POST | `/v1/claim/{claimNumber}/return-confirmation` | Confirm return receipt |
| 10 | GET | `/v1/units/{serial_number}/claim-history` | Past claims for a serial |
| 11 | POST | Webhook subscription (planned) | Register callback URL |

---

## 9. Error Handling

### Standard Error Response

```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable description",
  "details": {}
}
```

### HTTP Status Codes

| Status | Code | Description |
|--------|------|-------------|
| 400 | `INVALID_REQUEST` | Malformed or missing required fields |
| 401 | `UNAUTHORIZED` | Missing/expired token |
| 403 | `FORBIDDEN` | Valid token, insufficient permissions |
| 404 | `NOT_FOUND` | Resource doesn't exist (e.g., invalid serial) |
| 409 | `CONFLICT` | Duplicate operation (e.g., already registered) |
| 422 | `VALIDATION_ERROR` | Fails business validation |
| 429 | `RATE_LIMITED` | Too many requests (include `Retry-After` header) |
| 500 | `INTERNAL_ERROR` | Server-side failure |
| 503 | `SERVICE_UNAVAILABLE` | Backend system temporarily down |

---

## 10. Gap Analysis & Open Items

### Critical

| # | Gap | Impact | Status |
|---|-----|--------|--------|
| 1 | Warranty coverage tiers | ✅ Resolved — `coverages[]` array on entitlement response | Addressed |
| 2 | Installation date on unit lookup | ✅ Resolved — `InstallationInfo` on `UnitDetails` | Addressed |
| 3 | Registration write-back | ✅ Resolved — `POST /v1/registrations/sync` | Addressed |
| 4 | Claim history per serial | ✅ Resolved — `GET /v1/units/{sn}/claim-history` (Continuum-managed) | Addressed |
| 5 | Authorization number | 🔴 Open — No pre-approval reference for contractor-to-distributor exchange | Needs design |

### High

| # | Gap | Impact | Status |
|---|-----|--------|--------|
| 6 | Labor/travel rate tables | ✅ Resolved — `GET /v1/labor-rates` + entitlement response | Addressed |
| 7 | Parts return workflow | ✅ Resolved — return-instructions + return-confirmation endpoints | Addressed |
| 8 | Product brand | ✅ Resolved — `brand` field on `ProductInfo` | Addressed |
| 9 | Pro-rated coverage | ✅ Resolved — `ProRatedSchedule` on `WarrantyCoverage` | Addressed |
| 10 | Claim submission deadline | ✅ Resolved — `claimSubmissionDeadlineDays` on entitlement | Addressed |

### Moderate — Still Open

| # | Gap | Impact | Notes |
|---|-----|--------|-------|
| 11 | Contractor certification validation | Non-certified contractors may file claims that get denied | `isCertified` field exists; enforcement is TBD |
| 12 | Warranty transferability | Can't handle ownership changes | No transfer endpoint; `isTransferable` flag exists |
| 13 | Structured attachment types | Generic upload; manufacturer can't auto-validate completeness | `attachmentType` field added; typed validation TBD |
| 14 | Webhooks | Must poll for status; delayed updates | Planned, not yet implemented |
| 15 | Replacement unit warranty linking | Can't link replacement unit to original claim/coverage | Needs design |
| 16 | Proof of purchase requirements | No structured field for when required or what qualifies | `attachmentsRequired` exists; rules TBD |

---

*End of specification.*
