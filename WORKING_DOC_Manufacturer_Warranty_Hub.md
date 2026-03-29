# Manufacturer Warranty Hub — Working Document

**Status:** Draft / Working Analysis  
**Last Updated:** March 29, 2026  
**Purpose:** Capture the end-to-end understanding of the Manufacturer Warranty Hub process, the API surface needed, and identified data gaps.

---

## 1. What Is This?

Continuum is a warranty and returns automation platform. For the Manufacturer Warranty Hub, Continuum operates as the **consumer-facing warranty portal** on behalf of the manufacturer. Consumers (homeowners), contractors, and distributors interact with Continuum's UI. Behind the scenes, Continuum calls into the manufacturer's middleware API to retrieve data from and push updates to their systems of record (ERPs, PIMs, legacy databases — often green-screen terminal systems where manual data entry is the norm).

The manufacturer builds and maintains the middleware API layer. This documentation defines the contract they must implement so Continuum can do its job.

### 1.1 Why This Matters

Warranty processing in the manufacturing world — particularly in industries like water heaters, HVAC, and similar durable goods — is painfully manual. A typical claim might involve:

- A homeowner calling a toll-free number
- A phone agent looking up a serial number in a green-screen terminal
- Manually verifying warranty coverage by reading off dates
- Handwriting or typing claim details into legacy software
- Mailing physical claim forms to a claims department
- Waiting days or weeks for manual review

Continuum eliminates this by automating the data collection, lookups, entitlement checks, and claims submission — then pushing the results directly into the manufacturer's systems. But to do that, Continuum needs a well-defined, reliable API to call.

### 1.2 Key Actors

| Actor                             | Role                                                                                                   | System                                                  |
| --------------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------- |
| **Consumer / Homeowner**          | End user who owns the product. Registers products, checks warranty status, initiates service requests. | Continuum UI (web portal)                               |
| **Contractor / Service Provider** | Licensed professional who diagnoses failures, performs repairs, files warranty claims.                 | Continuum UI (contractor portal)                        |
| **Distributor**                   | Sells products to contractors/consumers. May also process warranty exchanges at the counter.           | Continuum UI + their own systems                        |
| **Manufacturer**                  | Makes the product. Owns warranty terms, parts inventory, claims processing, and payment.               | ERP, PIM, warranty database (behind the middleware API) |
| **Continuum**                     | Orchestrates the entire process. Provides the UI, automates data flows, calls manufacturer APIs.       | Warranty platform                                       |

### 1.3 Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────────┐     ┌──────────────────┐
│  Consumer /  │     │              │     │   Manufacturer's    │     │  Manufacturer's  │
│  Contractor  │────▶│  Continuum   │────▶│   Middleware API     │────▶│  Systems of      │
│  Browser     │◀────│  Platform    │◀────│   (YOU BUILD THIS)  │◀────│  Record          │
│              │     │              │     │                     │     │  (ERP/PIM/etc.)  │
└─────────────┘     └──────────────┘     └─────────────────────┘     └──────────────────┘
```

The middleware API is the integration boundary. Continuum defines the contract (this documentation). The manufacturer's IT team implements it, mapping their internal data to the agreed-upon schemas.

---

## 2. End-to-End Process Flows

### 2.1 Product Registration

The registration flow is the entry point. A consumer registers their product after purchase/installation. This establishes the warranty record and is often required before a claim can be filed.

**Steps:**

1. Consumer enters serial number on the Continuum registration portal.
2. Continuum calls the manufacturer API to **validate the serial number** and retrieve unit details (model, manufacture date, product info).
3. The portal auto-fills product fields from the API response.
4. Continuum calls the manufacturer API to **look up the distributor** that originally sold the unit (ship-to / sold-to from the sales transaction).
5. The portal auto-fills distributor fields.
6. Consumer enters their installation details (install date, address, installation type: residential / commercial / multi-family).
7. If a contractor performed the installation, Continuum calls the manufacturer API to **search the contractor directory** so the consumer can select their installer.
8. Consumer submits the registration.
9. Continuum stores the registration in its own database.
10. **[GAP]** Registration should also push back to the manufacturer's system (currently spec'd as platform-only with no write-back). Registration affects warranty terms — many manufacturers extend coverage for registered units or require registration to activate full warranty.

**Key Data Points Captured:**

- Serial number, model number
- Purchase date, installation date
- Installation type (residential, commercial, multi-family)
- Homeowner info (name, address, email, phone)
- Contractor info (company, license, contact)
- Distributor info (auto-filled from API)

**Warranty Effective Date Logic:**
The warranty start date is the **installation date** if documented (via registration). If no installation date is recorded, manufacturers typically default to **manufacture date + 90 days**. This makes registration critically important for both the consumer and the manufacturer.

---

### 2.2 Warranty Verification / Inquiry

A consumer, contractor, or distributor wants to check whether a unit is under warranty and what's covered.

**Steps:**

1. User enters serial number on the warranty verification page.
2. Continuum calls the manufacturer API to **retrieve unit details** — product info, sales data, manufacturing info, and warranty status.
3. Continuum calls the manufacturer API to **retrieve entitlement information** — the detailed warranty coverage for this specific unit.
4. The portal displays:
   - Product identification (model, description, serial)
   - Warranty status and coverage details
   - **[GAP]** Coverage should be broken out by tier: tank, parts, labor — each with their own status and expiration
   - Installation and registration status
   - Homeowner information (if registered)
5. If the user needs replacement parts, Continuum calls the manufacturer API to **retrieve the bill of materials** for the unit.
6. Continuum calls the manufacturer API (PIM) to **retrieve part images** for display.
7. The portal displays replacement parts with images, specs, and availability.

**Warranty Coverage — Real-World Complexity:**

Manufacturers like AO Smith and Rheem don't have a single "warranty expiration date." Coverage is tiered:

| Component             | Typical Coverage                            |
| --------------------- | ------------------------------------------- |
| Tank / Heat Exchanger | 6–15 years (varies by product tier)         |
| Component Parts       | 5–12 years                                  |
| Labor                 | 1–3 years (often only 1 year)               |
| Specific Components   | Varies (e.g., LIFEGUARD elements = 3 years) |

Additionally:

- Coverage may differ for **residential vs. commercial** installations
- Some coverage becomes **pro-rated** after a period (e.g., full replacement for years 1-5, pro-rated credit years 6-10)
- **Registration** may unlock extended coverage
- Warranty is typically **non-transferable** (original owner only, at original installation location)

---

### 2.3 Document Center

Consumers and contractors need access to product documentation — installation guides, service manuals, spec sheets, wiring diagrams, troubleshooting guides, safety instructions, and parts lists.

**Steps:**

1. User searches by model number.
2. Continuum calls the manufacturer API to **retrieve documents** associated with that model.
3. The portal displays documents organized by category with download links.
4. Additionally, manufacturer admins can **manually upload documents** through the platform for models where the API source doesn't have the content.

**Document Categories:**

- Installation Guide
- Service Manual
- Parts List
- Warranty Information
- Spec Sheet
- Wiring Diagram
- Troubleshooting Guide
- Safety Instructions

---

### 2.4 Warranty Claim Processing — The Core Flow

This is the primary automation target. A contractor diagnoses a failure on a consumer's unit, needs to get warranty coverage confirmed, obtain replacement parts, and file a claim for reimbursement.

**Real-World Process (What We're Automating):**

Today, without Continuum, a contractor might:

1. Call the manufacturer's 1-800 number
2. Wait on hold
3. Read a serial number to a phone agent
4. Agent types it into a green-screen terminal
5. Agent reads back warranty status
6. Agent issues an authorization number (P or R prefix at AO Smith)
7. Contractor goes to local distributor with authorization number
8. Distributor exchanges the part or unit
9. Distributor mails a paper claim form to the manufacturer
10. Manufacturer manually processes the claim weeks later
11. Credit memo eventually issued

**Automated Process via Continuum:**

#### Phase 1: Unit Identification & Entitlement Check

1. Contractor or distributor enters the serial number of the failed unit.
2. Continuum calls **Validate Unit** (`/v1/validate-unit`) — confirms the serial is valid, returns model number(s). If multiple models are associated (multi-component systems), user selects the correct one from a dropdown.
3. Continuum calls **Entitlement Information** (`/v1/entitlement-information`) — retrieves:
   - Warranty validity (is the serial under any active coverage?)
   - Installation type (residential / commercial / multi-family)
   - Installation date
   - Registration status
   - Homeowner details
   - Purchaser/distributor details
   - **[GAP]** Should return detailed coverage tiers (tank, parts, labor) with individual statuses and dates
   - **[GAP]** Should return applicable labor/travel rate allowances
   - **[GAP]** Should return claim submission deadline rules
   - **[GAP]** Should return whether this serial has prior claims (claim history)

#### Phase 2: Failure Details & Parts Selection

4. Continuum calls **Claim Types** (`/v1/claim-types`) — retrieves available claim types for this scenario (e.g., "Parts Only", "Full Unit Replacement", "Labor Claim").
5. Continuum calls **Bill of Materials** (`/v1/bill-of-materials`) — retrieves the BOM for the serial/model so the contractor can identify which part(s) failed.
6. Contractor selects the failed part(s) from the BOM.
7. For each failed part, Continuum calls **Warranty Codes** (`/v1/products/warranty-codes`) — retrieves the hierarchical tree of warranty disposition  codes (general reason → specific reason → detail).
8. Contractor selects the reason code(s) and adds any notes.
9. For each failed part, Continuum calls **Replacement Parts** (`/v1/products/replacements`) — finds suitable replacements, including superseded parts and non-OEM alternatives (if enabled).
10. Contractor selects replacement part(s).

#### Phase 3: Claim Submission

11. Contractor fills in remaining details:
    - Install date (if not already known)
    - Homeowner information
    - Contractor information
    - Labor hours and rate (if labor claim)
    - **[GAP]** Travel miles and rate (if travel reimbursement applies)
    - Notes / description of failure
12. Continuum calls **Validate Claim** (`/v1/claim/validate`) — pre-validates the claim before submission. This is a dry run that checks:
    - Entitlement is valid
    - Parts are covered
    - Reason codes are valid
    - Required fields are complete
    - **[GAP]** Should return any required documentation/attachments list
13. If validation passes, Continuum calls **Create Claim** (`POST /v1/claim`) — submits the warranty claim.
14. The response includes:
    - Claim number (the manufacturer's reference)
    - Whether the failed part is requested for return
    - **[GAP]** Authorization number (the reference the contractor carries to the distributor)
    - Financial details (unit price, labor allowance, tax, markup, total claim amount)
    - **[GAP]** Return shipping instructions if part return is required
    - Whether attachments are required (and a prompt describing what's needed)

#### Phase 4: Attachments & Documentation

15. If attachments are required (e.g., photo of rating plate, proof of purchase, photo of failed part), Continuum calls **Upload Attachment** (`POST /v1/claim/upload-attachment`) with the claim number and file(s).
16. **[GAP]** The API should support structured attachment types (rating_plate_photo, proof_of_purchase, failed_part_photo, installation_photo) rather than generic binary uploads, so the manufacturer's system can validate completeness.

#### Phase 5: Claim Tracking & Resolution

17. Continuum periodically calls **Claim Status** (`GET /v1/claim/status`) to check the claim's progress.
18. Possible statuses: `PENDING`, `APPROVED`, `DENIED`, `NEED_CORRECTIONS`, `NEED_SERIAL_NUMBER`, `RETURN_PRODUCT`, `PRODUCT_COLLECTED`, `READY_FOR_PICKUP`, `PICKED_UP`, `PAID`, `CANCELLED`, `ERROR`.
19. If `NEED_CORRECTIONS` — Continuum notifies the contractor and they update the claim via **Update Claim** (`PATCH /v1/claim`).
20. If `RETURN_PRODUCT` — the contractor ships the failed part back. **[GAP]** Need return tracking, return deadline, and receipt confirmation endpoints.
21. If `APPROVED` → `PAID` — the manufacturer issues credit. The distributor receives a credit memo.
22. **[GAP]** Webhooks/callbacks for async status notifications would eliminate polling and improve responsiveness.

#### Phase 6: Claim Closure

23. Claim reaches terminal state (`PAID`, `DENIED`, `CANCELLED`).
24. **[GAP]** If the claim resulted in a full unit replacement, the replacement unit's warranty should be linked back (warranted only for the unexpired portion of the original).

---

### 2.5 Historical Data Import

A one-time (or recurring) job to pull existing warranty registration data from the manufacturer's legacy systems into Continuum so the platform starts with a complete history.

**Steps:**

1. Continuum calls **Import Registrations** (`GET /api/v1/import/registrations`) with pagination.
2. Supports incremental sync via the `since` parameter (only records after a given date).
3. Continuum stores imported records in its database.
4. Runs as a scheduled job until fully synced, then switches to incremental mode.

---

## 3. API Surface — High-Level Inventory

### 3.1 Authentication

| Endpoint           | Method | Purpose                                        |
| ------------------ | ------ | ---------------------------------------------- |
| `/v1/oauth2/token` | POST   | Exchange client credentials for a bearer token |

All subsequent API calls use the bearer token in the `Authorization` header.

---

### 3.2 Unit & Product Data APIs

These endpoints retrieve product, warranty, and manufacturing data from the manufacturer's systems.

| Endpoint                                                 | Method | Purpose                                                                   | Source System     |
| -------------------------------------------------------- | ------ | ------------------------------------------------------------------------- | ----------------- |
| `/v1/validate-unit`                                      | GET    | Validate serial number, return model(s)                                   | ERP               |
| `/v1/entitlement-information`                            | GET    | Detailed warranty coverage, homeowner info, registration status           | ERP / Warranty DB |
| `/v1/units/{serial_number}`                              | GET    | Full unit record: product, sales, manufacturing, warranty                 | ERP               |
| `/v1/units/{serial_number}/parts`                        | GET    | BOM / replacement parts with specs and images                             | ERP + PIM         |
| `/v1/units/{serial_number}/distributor`                  | GET    | Distributor that sold the unit (ship-to/sold-to)                          | ERP               |
| `/v1/pim/images/{part_number}`                           | GET    | Part images from PIM system                                               | PIM               |
| **[NEW]** `/v1/units/{serial_number}/warranty-coverages` | GET    | Detailed coverage tiers (tank, parts, labor) with individual status/dates | ERP / Warranty DB |
| **[NEW]** `/v1/units/{serial_number}/claim-history`      | GET    | Past claims filed against this serial                                     | Continuum         |

---

### 3.3 Reference Data APIs

Lookup tables and directories that power form dropdowns and selections.

| Endpoint                      | Method | Purpose                                                               | Source System     |
| ----------------------------- | ------ | --------------------------------------------------------------------- | ----------------- |
| `/v1/claim-types`             | GET    | Available claim types                                                 | Continuum         |
| `/v1/products/warranty-codes` | GET    | Hierarchical warranty codes for a part                                | ERP / Warranty DB |
| `/v1/products/replacements`   | GET    | Replacement part options (incl. superseded, non-OEM)                  | ERP / PIM         |
| `/v1/contractors`             | GET    | Contractor directory search (by zip, name, certification)             | ERP               |
| `/v1/locations`               | GET    | Registered warranty locations                                         | ERP               |
| `/v1/corporations`            | GET    | Corporation list (authorized distributors)                            | ERP               |
| `/v1/corporations/details`    | GET    | Corporation details                       | ERP               |
| `/v1/corporations/customers`  | GET    | Customers under a distributor                                         | ERP               |
| **[NEW]** `/v1/labor-rates`   | GET    | Applicable labor/travel rate schedule by region and installation type | Warranty DB       |

---

### 3.4 Document APIs

| Endpoint        | Method | Purpose                                                  | Source System       |
| --------------- | ------ | -------------------------------------------------------- | ------------------- |
| `/v1/documents` | GET    | Fetch product docs by model (manuals, spec sheets, etc.) | PIM / DAM           |
| `/v1/documents` | POST   | Manual document upload (multipart/form-data)             | Continuum → Storage |

---

### 3.5 Claims Processing APIs

The core transactional endpoints for warranty claims.

| Endpoint                                                | Method | Purpose                                                 | Source System |
| ------------------------------------------------------- | ------ | ------------------------------------------------------- | ------------- |
| `/v1/claim/validate`                                    | POST   | Pre-validate a claim before submission                  | Continuum     |
| `/v1/claim`                                             | POST   | Create a new warranty claim                             | Continuum     |
| `/v1/claim`                                             | PATCH  | Update an existing claim (corrections, additional info) | Continuum     |
| `/v1/claim`                                             | DELETE | Cancel/delete a warranty claim                          | Continuum     |
| `/v1/claim/status`                                      | GET    | Check claim status                                      | Continuum     |
| `/v1/claim/upload-attachment`                           | POST   | Upload supporting documents/photos                      | Continuum     |
| **[NEW]** `/v1/claim/{claimNumber}/return-instructions` | GET    | Get return shipping details, deadlines, labels          | Continuum     |
| **[NEW]** `/v1/claim/{claimNumber}/return-confirmation` | POST   | Confirm return receipt by manufacturer                  | Continuum     |

---

### 3.6 Registration APIs

| Endpoint                           | Method | Purpose                                                 | Source System     |
| ---------------------------------- | ------ | ------------------------------------------------------- | ----------------- |
| `/v1/registrations`                | POST   | Register a product (currently platform-only)            | Continuum         |
| `/v1/import/registrations`         | GET    | Bulk historical registration import (paginated)         | ERP / Legacy DB   |
| **[NEW]** `/v1/registrations/sync` | POST   | Push registration to manufacturer's system (write-back) | ERP / Warranty DB |

---

### 3.7 Webhooks / Notifications (New)

| Endpoint                              | Method | Purpose                                                    | Direction                |
| ------------------------------------- | ------ | ---------------------------------------------------------- | ------------------------ |
| **[NEW]** Webhook subscription config | POST   | Register Continuum's callback URL for status change events | Manufacturer → Continuum |
| **[NEW]** Webhook payload             | POST   | Manufacturer pushes claim status changes to Continuum      | Manufacturer → Continuum |

---

## 4. Identified Data Gaps — Prioritized

### Critical (Blocks core functionality or causes incorrect claim outcomes)

| #   | Gap                                                                                                | Impact                                                                                | Where It Applies                  |
| --- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | --------------------------------- |
| 1   | **Warranty coverage tiers** — No multi-tier coverage (tank/parts/labor with separate dates)        | Claims may be filed for components no longer covered; incorrect entitlement decisions | Entitlement response, Unit lookup |
| 2   | **Installation date on unit lookup** — Only manufacture_date returned, no install_date             | Warranty effective date calculated incorrectly                                        | Unit lookup response              |
| 3   | **Registration write-back** — Registration is platform-only, no sync to manufacturer               | Manufacturer's system doesn't know the unit is registered; may affect coverage terms  | Registration flow                 |
| 4   | **Claim history per serial** — No endpoint to retrieve past claims                                 | Duplicate claims, no audit trail, no repeat-failure detection                         | Claims flow                       |
| 5   | **Authorization number** — No pre-approval reference for the contractor to take to the distributor | Breaks the real-world exchange process at the distributor counter                     | Claim creation response           |

### High (Significant functionality or data quality issues)

| #   | Gap                                                                                                            | Impact                                                               | Where It Applies         |
| --- | -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- | ------------------------ |
| 6   | **Labor/travel rate tables** — No endpoint to retrieve applicable allowance schedules                          | Contractors can't see what they'll be reimbursed before filing       | Pre-claim, entitlement   |
| 7   | **Parts return workflow** — No return instructions, deadlines, tracking, or receipt confirmation               | Part returns can't be managed through the platform                   | Post-claim processing    |
| 8   | **Product brand** — No brand field on unit data (manufacturer may own multiple brands with different terms)    | Wrong warranty terms applied if brands have different coverage       | Unit lookup, entitlement |
| 9   | **Pro-rated coverage** — No concept of full vs. pro-rated warranty periods                                     | Incorrect credit calculations                                        | Entitlement, claims      |
| 10  | **Claim submission deadline** — No field or validation for "claims must be submitted within X days of failure" | Claims accepted that should be rejected; manufacturer audit failures | Claims validation        |

### Moderate (Important for completeness and edge cases)

| #   | Gap                                                                                                | Impact                                                            | Where It Applies          |
| --- | -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------- |
| 11  | **Fuel type / product category** — Not explicit in unit data                                       | Affects which warranty terms, parts, and reason codes apply       | Unit lookup               |
| 12  | **Contractor certification validation** — Directory doesn't verify manufacturer-specific training  | Non-certified contractors file claims that get denied             | Contractor search         |
| 13  | **Warranty transferability** — No endpoint or status for warranty transfers                        | Can't handle ownership changes                                    | Registration, entitlement |
| 14  | **Structured attachment types** — Upload is generic binary, no typed categories                    | Manufacturer can't auto-validate document completeness            | Attachment upload         |
| 15  | **Webhooks** — No async notification mechanism for claim status changes                            | Continuum must poll; delayed status updates in portal             | Claim tracking            |
| 16  | **Replacement unit warranty linking** — No way to link replacement unit to original claim/coverage | Replacement warranty terms can't be properly calculated           | Post-claim                |
| 17  | **Proof of purchase requirements** — No structured field for when it's required or what qualifies  | Claims denied for missing documentation the system didn't request | Claims validation         |

---

## 5. Data Model Observations

### 5.1 Core Entity Alignment

The V2 Core entities (`primary-data-types.json`) establish good patterns that the warranty hub endpoints should follow:

- **`extendedAttributes[]`** — The `DataAttribute` pattern (name/value/section) is excellent for manufacturer-specific custom fields. Every warranty hub response should support this.
- **`CoreAccountLocation`** — The composition pattern (account context wrapping a location) is well-suited for ship-to/return-to addresses in claims.
- **`CoreOrganization`** — The unified entity with `organizationType` string (Customer, Vendor, Supplier) is the right approach. The warranty hub's separate Corporation/Customer models could benefit from alignment here.
- **`CoreLineShipment`** — This is exactly what's needed for the parts-return tracking gap.
- **String-typed enums** — The V2 pattern of using strings rather than fixed enums for classifications (account type, location type, org type) is the right call for ERP-agnostic middleware. The warranty hub should follow this pattern more consistently.

### 5.2 Warranty Claims Model — Observations

The current `CreateWarrantyDto_Request` and response are heavily denormalized — lots of flat fields directly on the response (address, name, financial details). This works but could benefit from:

- Grouping financial fields into a `financials` or `settlement` object
- Using the `CoreAddress` pattern for address fields instead of flat `addr1`, `city`, `state`, `zipCode`
- Adding a `coverages[]` array to entitlement responses
- Adding `brand` to unit/entitlement data
- Structured `authorization` object on claim response (number, type, expiration)

---

## 6. Next Steps

1. **Validate this document** with the team — confirm or correct the process understanding.
2. **Prioritize the gaps** — which ones block launch vs. which can be deferred.
3. **Begin documentation portal build** — structure the Mintlify site around the three flows, with the Manufacturer Warranty Hub as the primary focus.
4. **Flesh out endpoint specs** — take the high-level API inventory above and build detailed endpoint documentation with request/response schemas, error codes, and examples.
5. **Create implementation guide** — a step-by-step guide for a manufacturer's IT team to build the middleware.

---

*This is a living document. It will be updated as we refine the specifications and build out the documentation portal.*
