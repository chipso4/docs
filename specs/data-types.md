# Manufacturer Warranty Hub — Shared Data Types

**Version:** 1.2-draft  
**Last Updated:** March 29, 2026  

These are the shared data types used across all manufacturer-facing API endpoints. Manufacturers implementing the middleware API should use these schemas as the canonical reference for request and response structures.

---

## Design Principles

1. **ERP-agnostic:** All classification fields (types, statuses, categories) are `string` rather than fixed enums unless explicitly constrained. This allows each manufacturer to map their own internal values regardless of their ERP system (SAP, Oracle, NetSuite, Epicor, custom, etc.).
2. **Nullable by default:** All fields are nullable unless marked as `required`. This accommodates ERP systems that may not populate every field.
3. **Extensible:** Every entity supports `extendedAttributes[]` for manufacturer-specific custom fields that don't fit the standard schema. This is the primary mechanism for industry-specific and product-specific data.
4. **Consistent patterns:** Address, contact, and location structures are reused across all endpoints following the same shape.
5. **Multi-identifier parts:** Parts are identified by their primary part number (`partNumber`), with additional identifiers (superseded part numbers, brand-specific cross-references) exposed via `crossReferences`.
6. **Unified media:** Documents, images, manuals, and other media are accessed through a single media model, retrievable by part number, model number, or media ID.

---

## Core Value Objects

### Address

A standard address structure used wherever physical addresses appear.

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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `address1` | string | No | Primary street address |
| `address2` | string | No | Secondary address line (suite, unit, etc.) |
| `address3` | string | No | Additional address line |
| `city` | string | No | City name |
| `state` | string | No | State or province full name |
| `stateCode` | string | No | State or province abbreviation (e.g., "TX", "ON") |
| `zip` | string | No | Postal / zip code |
| `country` | string | No | Country full name |
| `countryCode` | string | No | ISO 3166-1 alpha-2 country code (e.g., "US", "CA") |

---

### DataAttribute

A generic name-value pair for extending any entity with custom manufacturer-specific data. This is the primary extensibility mechanism — any data that doesn't fit the standard schema goes here, including product specifications, labor rates, claim-specific fields, and industry-specific data.

```json
{
  "name": "string",
  "value": "string | null",
  "section": "string | null"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | **Yes** | Attribute name (e.g., "CapacityGallons", "Voltage", "LaborRate") |
| `value` | string | No | Attribute value |
| `section` | string | No | Logical grouping/category for the attribute (e.g., "Electrical", "Capacity", "Efficiency") |

**Usage:** Every response entity includes an `extendedAttributes` array. Manufacturers should use this for:
- **Product specifications** — technical details specific to their product category
- **Industry-specific data** — fields unique to water heaters, HVAC, appliances, etc.
- **ERP-specific fields** — internal classification codes, material groups, etc.
- **Regional data** — state-specific warranty terms, compliance codes, etc.

---

## Contact Types

### HomeownerInfo

Represents the end consumer / homeowner who owns the product.

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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | No | Manufacturer's internal homeowner ID (if previously registered) |
| `businessName` | string | No | Business name (for commercial installations) |
| `firstName` | string | No | Homeowner first name |
| `lastName` | string | No | Homeowner last name |
| `email` | string | No | Email address |
| `phone` | string | No | Phone number |
| `locationId` | string | No | Reference to a location record in the manufacturer's system |
| `address` | Address | No | Homeowner's address (uses standard Address object) |

---

### ContractorInfo

Represents the licensed professional who installs or services the product.

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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | No | Manufacturer's internal contractor ID |
| `firstName` | string | No | Contact first name |
| `lastName` | string | No | Contact last name |
| `companyName` | string | No | Business / company name |
| `email` | string | No | Email address |
| `phone` | string | No | Phone number |
| `licenseNumber` | string | No | Contractor's trade license number |
| `isCertified` | boolean | No | Whether the contractor is certified by this manufacturer for warranty work |
| `certifications` | string[] | No | List of manufacturer-specific certifications held |
| `locationId` | string | No | Reference to a location record |
| `address` | Address | No | Contractor's business address |

---

### DistributorInfo

Represents the distributor / wholesaler that sold the unit.

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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `distributorId` | string | No | Manufacturer's internal distributor ID |
| `name` | string | No | Distributor business name |
| `accountNumber` | string | No | Distributor's account number with the manufacturer |
| `purchaseDate` | date | No | Date the distributor purchased/received the unit |
| `address` | Address | No | Distributor's address |
| `phone` | string | No | Phone number |
| `email` | string | No | Email address |

---

### ClaimContactInfo

The person submitting or managing a warranty claim (may be different from the homeowner or contractor).

```json
{
  "id": "string | null",
  "firstName": "string | null",
  "lastName": "string | null",
  "email": "string | null",
  "phone": "string | null"
}
```

---

## Product & Unit Types

### UnitDetails

The complete record for a manufactured unit, identified by serial number. This is the primary entity for the warranty hub.

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

---

### ProductInfo

Product identification and classification data. Technical specifications are carried in `extendedAttributes` to support any industry and product type.

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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `baseModel` | string | No | Base model number (e.g., "EN6-50-DORT") |
| `modelDescription` | string | No | Full model description |
| `sku` | string | No | SKU / item number |
| `materialNumber` | string | No | ERP material/item number |
| `materialDescription` | string | No | ERP material description |
| `brand` | string | No | Product brand (e.g., "Rheem", "Ruud", "Richmond"). A manufacturer may own multiple brands with different warranty terms. |
| `productLine` | string | No | Product line grouping (e.g., "Performance Platinum", "Professional Classic") |
| `seriesName` | string | No | Series identifier within a product line |
| `productCategory` | string | No | Product category (e.g., "Water Heater", "Air Conditioner", "Furnace", "Boiler") |
| `productType` | string | No | More specific product type (e.g., "Tank", "Tankless", "Heat Pump", "Package Unit") |
| `fuelType` | string | No | Energy source (e.g., "Gas", "Electric", "Solar", "Hybrid", "LP") |
| `extendedAttributes` | DataAttribute[] | No | **Product technical specifications and any industry-specific data.** See examples below. |

#### Example: Water Heater Technical Specs via `extendedAttributes`

```json
"extendedAttributes": [
  { "name": "CapacityGallons", "value": "50", "section": "Capacity" },
  { "name": "Voltage", "value": "240V", "section": "Electrical" },
  { "name": "Phase", "value": "1", "section": "Electrical" },
  { "name": "Hertz", "value": "60Hz", "section": "Electrical" },
  { "name": "UEF", "value": "0.93", "section": "Efficiency" },
  { "name": "FirstHourRating", "value": "67 gal", "section": "Performance" },
  { "name": "RecoveryRate", "value": "21 GPH", "section": "Performance" },
  { "name": "VentingType", "value": "Atmospheric", "section": "Configuration" }
]
```

#### Example: HVAC Unit Technical Specs via `extendedAttributes`

```json
"extendedAttributes": [
  { "name": "CapacityBTUH", "value": "36000", "section": "Capacity" },
  { "name": "CapacityTons", "value": "3", "section": "Capacity" },
  { "name": "Voltage", "value": "208/230V", "section": "Electrical" },
  { "name": "Phase", "value": "1", "section": "Electrical" },
  { "name": "SEER2", "value": "15.2", "section": "Efficiency" },
  { "name": "EER", "value": "12.5", "section": "Efficiency" },
  { "name": "RefrigerantType", "value": "R-410A", "section": "Refrigerant" },
  { "name": "RefrigerantChargeOz", "value": "79", "section": "Refrigerant" },
  { "name": "CompressorType", "value": "Scroll", "section": "Compressor" },
  { "name": "TotalCompressors", "value": "1", "section": "Compressor" }
]
```

> **Note for manufacturers:** The `section` field enables grouping in the UI. Use consistent section names across your product catalog (e.g., always "Electrical" not sometimes "Electrical" and sometimes "Power"). Continuum will render these as grouped attribute tables.

---

### ManufacturingInfo

Manufacturing origin data.

```json
{
  "plantCode": "string | null",
  "plantName": "string | null",
  "manufactureDate": "date | null",
  "lotNumber": "string | null",
  "countryOfOrigin": "string | null"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `plantCode` | string | No | Manufacturing plant identifier |
| `plantName` | string | No | Manufacturing plant name |
| `manufactureDate` | date | No | Date the unit was manufactured |
| `lotNumber` | string | No | Production lot/batch number |
| `countryOfOrigin` | string | No | Country where the unit was manufactured |

---

### SalesInfo

Sales transaction data for how the unit reached the distribution channel.

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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `salesOrg` | string | No | Sales organization code |
| `salesOrderNumber` | string | No | Original sales order number |
| `originalPoNumber` | string | No | Customer's purchase order number |
| `shippingPlant` | string | No | Plant the unit shipped from |
| `customerId` | string | No | Distributor/customer account ID in the ERP |
| `customerName` | string | No | Distributor/customer name |
| `shipDate` | date | No | Date the unit shipped |
| `invoiceNumber` | string | No | Invoice number for the original sale |
| `invoiceDate` | date | No | Invoice date |

---

### InstallationInfo

Installation details — typically populated from product registration.

```json
{
  "installDate": "date | null",
  "installationType": "string | null",
  "installedBy": { /* ContractorInfo */ },
  "installationAddress": { /* Address */ },
  "isProperlyDocumented": "boolean | null"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `installDate` | date | No | Date the unit was installed. **This is the warranty effective date if documented.** If null, warranty effective date defaults to `manufactureDate + 90 days`. |
| `installationType` | string | No | Installation context: `"Residential"`, `"Commercial"`, `"MultiFamily"`. Affects warranty terms. |
| `installedBy` | ContractorInfo | No | The contractor who performed the installation |
| `installationAddress` | Address | No | Where the unit is installed |
| `isProperlyDocumented` | boolean | No | Whether the installation date is properly documented (affects warranty effective date calculation) |

---

### RegistrationInfo

Registration status and metadata.

```json
{
  "isRegistered": "boolean",
  "registrationDate": "datetime | null",
  "registrationSource": "string | null",
  "registeredOwner": { /* HomeownerInfo */ },
  "affectsWarrantyTerms": "boolean | null",
  "extendedCoverageDays": "integer | null"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `isRegistered` | boolean | **Yes** | Whether the unit has been registered |
| `registrationDate` | datetime | No | When the unit was registered |
| `registrationSource` | string | No | How the unit was registered (e.g., "web_portal", "mail_in", "dealer", "continuum") |
| `registeredOwner` | HomeownerInfo | No | The registered owner |
| `affectsWarrantyTerms` | boolean | No | Whether registration extends or unlocks additional warranty coverage |
| `extendedCoverageDays` | integer | No | Additional days of coverage granted by registration (if applicable) |

---

## Warranty Types

### WarrantySummary

High-level warranty status for quick display. For detailed tier-level coverage, use `WarrantyCoverage[]`.

```json
{
  "overallStatus": "string",
  "effectiveDate": "date | null",
  "isTransferable": "boolean | null",
  "currentOwnerIsOriginal": "boolean | null"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `overallStatus` | string | **Yes** | Aggregate warranty status. Recommended values: `"produced"`, `"active"`, `"partial"` (some tiers expired), `"expired"`, `"voided"`, `"pending"` |
| `effectiveDate` | date | No | The calculated warranty effective date (install date if documented, otherwise manufacture date + 90 days) |
| `isTransferable` | boolean | No | Whether this warranty can be transferred to a new owner |
| `currentOwnerIsOriginal` | boolean | No | Whether the current registered owner is the original purchaser |

#### Enum guidance: `overallStatus`

| Value | Description |
|-------|-------------|
| `produced` | Manufactured but not yet installed/registered/activated |
| `active` | At least one coverage tier is active |
| `partial` | Some coverage tiers have expired while others remain active (e.g., labor expired but parts still covered) |
| `expired` | All coverage tiers have expired |
| `voided` | Warranty has been voided (improper installation, unauthorized modifications, etc.) |
| `pending` | Registration received, warranty activation in progress |

---

### WarrantyCoverage

A single tier of warranty coverage. A unit will typically have multiple coverages (tank, parts, labor, etc.).

```json
{
  "coverageType": "string",
  "coverageLabel": "string | null",
  "status": "string",
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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `coverageType` | string | **Yes** | Coverage tier identifier. Common values: `"tank"`, `"heat_exchanger"`, `"parts"`, `"labor"`, `"component_specific"` |
| `coverageLabel` | string | No | Human-readable label (e.g., "Tank & Heat Exchanger", "Component Parts", "Labor") |
| `status` | string | **Yes** | Coverage status: `"active"`, `"expired"`, `"voided"`, `"pending"`, `"pro_rated"` |
| `effectiveDate` | date | No | When this coverage began |
| `expirationDate` | date | No | When this coverage expires |
| `coverageTermMonths` | integer | No | Original coverage term in months |
| `isProRated` | boolean | No | Whether this tier uses pro-rated reimbursement after a certain period |
| `proRatedSchedule` | ProRatedSchedule | No | The pro-ration schedule, if applicable |
| `coveredComponents` | string[] | No | Specific parts/components covered under this tier |
| `exclusions` | string[] | No | Known exclusions for this tier |
| `requiresRegistration` | boolean | No | Whether this tier requires product registration to be active |
| `extendedAttributes` | DataAttribute[] | No | Additional manufacturer-specific coverage data |

---

### ProRatedSchedule

Defines how reimbursement decreases over time for pro-rated coverage.

```json
{
  "fullCoverageEndDate": "date | null",
  "fullCoverageMonths": "integer | null",
  "tiers": [
    {
      "fromMonth": "integer",
      "toMonth": "integer",
      "reimbursementPercent": "number",
      "description": "string | null"
    }
  ]
}
```

**Example:**
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

---

## Part Identification & Cross-Reference

### Part Number Identity Model

Parts are identified by their **primary part number** (`partNumber`) — the manufacturer's canonical part number (MPN). Since manufacturers frequently supersede parts or sell the same part under different brand names, a **cross-reference** mechanism exposes these relationships.

Common cross-reference scenarios for a manufacturer:

| Scenario | Example |
|----------|---------|
| **Superseded part** — This part replaces an older part number | `SP20042` supersedes `SP20041` |
| **Brand cross-reference** — Same part sold under different brand names | Rheem `SP20042` = Ruud `SP20042R` = Richmond `SP20042-RC` |
| **OEM cross-reference** — The component maker's own part number | Rheem `SP20042` = Honeywell `WV8840A1000` |
| **UPC / GTIN** — Universal barcode identifier | `020352330426` |

---

### PartCrossReference

A single cross-reference entry linking the primary part number to an alternate identifier.

```json
{
  "referenceType": "string",
  "partNumber": "string",
  "brand": "string | null",
  "description": "string | null"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `referenceType` | string | **Yes** | Type of cross-reference. Common values: `"supersedes"`, `"superseded_by"`, `"brand_xref"`, `"oem_xref"`, `"upc"`, `"equivalent"` |
| `partNumber` | string | **Yes** | The cross-referenced part number or identifier |
| `brand` | string | No | The brand this cross-reference belongs to (relevant for `"brand_xref"`) |
| `description` | string | No | Description or notes about this cross-reference (e.g., "Improved corrosion resistance" for a supersession) |

---

## Parts & BOM Types

### BOMPart

A single part in a bill of materials for a unit.

```json
{
  "partNumber": "string",
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
  "extendedAttributes": [ /* DataAttribute[] */ ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `partNumber` | string | **Yes** | Manufacturer's canonical part number (MPN) |
| `description` | string | No | Human-readable part description |
| `crossReferences` | PartCrossReference[] | No | Supersedes, brand cross-references, OEM cross-references, UPCs |
| `quantity` | number | No | Quantity used in this unit's BOM |
| `unitOfMeasure` | string | No | Unit of measure |
| `partType` | string | No | Top-level part classification (e.g., "compressor", "motor", "filter", "thermostat", "anode", "element", "valve", "sensor", "control_board") |
| `partSubType` | string | No | Specific sub-classification (e.g., "scroll_compressor", "condenser_motor", "blower_motor") |
| `brand` | string | No | Part brand (may differ from unit brand for third-party components) |
| `isServiceable` | boolean | No | Whether this part can be individually serviced/replaced in the field |
| `isWarrantyCovered` | boolean | No | Whether this specific part is currently covered under warranty for this unit |
| `warrantyExpirationDate` | date | No | When warranty coverage for this part expires |
| `inStock` | boolean | No | Whether the part is currently available |
| `images` | MediaItem[] | No | Part images — inline media items (see Media Types). For full media including manuals and diagrams, use the `/v1/media` endpoint. |
| `notes` | string | No | Additional notes about this part in the context of this BOM |
| `replacementsAvailable` | integer | No | Count of available replacement options |
| `extendedAttributes` | DataAttribute[] | No | **Part specifications and any additional data.** See examples below. |

#### Example: Anode Rod Specs via `extendedAttributes`

```json
"extendedAttributes": [
  { "name": "Dimensions", "value": "29.0\" × 0.75\" × 0.75\"", "section": "Physical" },
  { "name": "Weight", "value": "1.25 lbs", "section": "Physical" },
  { "name": "MaterialType", "value": "Aluminum", "section": "Physical" },
  { "name": "ThreadSize", "value": "3/4\" NPT", "section": "Physical" },
  { "name": "Compatibility", "value": "EN6-50-DORT series", "section": "Fitment" },
  { "name": "InstallationNotes", "value": "Professional installation recommended. Requires 1-1/16\" socket.", "section": "Service" }
]
```

#### Example: Motor Specs via `extendedAttributes`

```json
"extendedAttributes": [
  { "name": "MotorType", "value": "PSC", "section": "Motor" },
  { "name": "HP", "value": "1/5", "section": "Motor" },
  { "name": "RPM", "value": "1075", "section": "Motor" },
  { "name": "Voltage", "value": "460V", "section": "Electrical" },
  { "name": "Phase", "value": "1", "section": "Electrical" },
  { "name": "FLA", "value": "0.6A", "section": "Electrical" },
  { "name": "ShaftDiameter", "value": "1/2\"", "section": "Physical" },
  { "name": "EnclosureType", "value": "TENV", "section": "Physical" }
]
```

---

### ReplacementPart

A replacement option for a failed part. May include the original part, superseded parts, and non-OEM alternatives.

```json
{
  "partNumber": "string",
  "description": "string | null",
  "crossReferences": [ /* PartCrossReference[] */ ],
  "brand": "string | null",
  "isOEM": "boolean",
  "replacementType": "string | null",
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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `partNumber` | string | **Yes** | Replacement part number |
| `description` | string | No | Part description |
| `crossReferences` | PartCrossReference[] | No | Supersession and brand cross-references |
| `brand` | string | No | Brand name |
| `isOEM` | boolean | **Yes** | Whether this is an OEM (original equipment manufacturer) part |
| `replacementType` | string | No | Type of replacement: `"exact"`, `"superseded"`, `"compatible"`, `"aftermarket"`, `"universal"` |
| `inStock` | boolean | No | Current availability |
| `leadTimeDays` | integer | No | Estimated lead time if not in stock |
| `listPrice` | number | No | Standard list/retail price |
| `warrantyPrice` | number | No | Price applicable under warranty claim |
| `score` | integer | No | Recommendation score (higher = better match) |
| `scoreDetail` | string | No | Explanation of the score |
| `images` | MediaItem[] | No | Part images |
| `extendedAttributes` | DataAttribute[] | No | Part specifications and additional data |

---

## Warranty Code Types

### WarrantyCode

A hierarchical warranty disposition code used to classify the reason for failure. These are typically organized in a tree structure (general → specific → detail).

```json
{
  "id": "string",
  "description": "string | null",
  "fullPath": "string | null",
  "idPath": "string | null",
  "requiresSerialNumber": "boolean | null",
  "requiresAttachment": "boolean | null",
  "children": [ /* WarrantyCode[] */ ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **Yes** | Unique code identifier |
| `description` | string | No | Human-readable description of this code |
| `fullPath` | string | No | Full path description (e.g., "Electrical > Thermostat > Failed to regulate") |
| `idPath` | string | No | Full path of IDs (e.g., "ELEC/THERM/FAIL-REG") |
| `requiresSerialNumber` | boolean | No | Whether selecting this code requires the failed part's serial number |
| `requiresAttachment` | boolean | No | Whether selecting this code requires a supporting attachment (photo, document) |
| `children` | WarrantyCode[] | No | Child codes (next level of specificity). Empty array or null = leaf node. |

---

## Media Types

### MediaItem

A unified media type covering documents, images, manuals, diagrams, and any other file associated with a product or part. All media is accessible through a single endpoint and can be linked to models, parts, or both.

```json
{
  "id": "string",
  "title": "string | null",
  "filename": "string | null",
  "mediaType": "string",
  "category": "string | null",
  "url": "string",
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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **Yes** | Unique media identifier (used to fetch individual items via `/v1/media/{id}`) |
| `title` | string | No | Human-readable title |
| `filename` | string | No | Original filename |
| `mediaType` | string | **Yes** | Top-level media type: `"document"`, `"image"`, `"video"`, `"diagram"` |
| `category` | string | No | Specific category within the media type (see enum below) |
| `url` | string | **Yes** | URL to access/download the media |
| `previewUrl` | string | No | Thumbnail or preview image URL |
| `mimeType` | string | No | MIME type (e.g., `"application/pdf"`, `"image/jpeg"`, `"image/png"`) |
| `fileSizeBytes` | integer | No | File size in bytes |
| `pageCount` | integer | No | Number of pages (for documents) |
| `width` | integer | No | Width in pixels (for images) |
| `height` | integer | No | Height in pixels (for images) |
| `lastUpdated` | date | No | When the media was last updated |
| `language` | string | No | Language (ISO 639-1, e.g., "en", "es") |
| `associatedModels` | string[] | No | Model numbers this media applies to |
| `associatedPartNumbers` | string[] | No | Part numbers this media applies to |
| `extendedAttributes` | DataAttribute[] | No | Additional metadata |

#### Enum guidance: `category`

**Document categories:**

| Value | Label |
|-------|-------|
| `installation_guide` | Installation Guide |
| `service_manual` | Service Manual |
| `parts_list` | Parts List |
| `warranty_information` | Warranty Information |
| `spec_sheet` | Spec Sheet |
| `wiring_diagram` | Wiring Diagram |
| `troubleshooting_guide` | Troubleshooting Guide |
| `safety_instructions` | Safety Instructions |
| `owners_manual` | Owner's Manual |
| `energy_guide` | Energy Guide |
| `dimensional_drawing` | Dimensional Drawing |
| `product_data` | Product Data Sheet |
| `service_bulletin` | Service / Technical Bulletin |
| `iom` | Installation, Operation & Maintenance |

**Image categories:**

| Value | Label |
|-------|-------|
| `product_photo` | Product Photo |
| `part_photo` | Part Photo |
| `installation_photo` | Installation Photo |
| `diagram` | Technical Diagram |
| `exploded_view` | Exploded View |
| `rating_plate` | Rating Plate Image |

---

## Location & Organization Types

### WarrantyLocation

A registered warranty service location (warehouse, branch, service center).

```json
{
  "locationId": "string",
  "locationName": "string | null",
  "locationType": "string | null",
  "address": { /* Address */ },
  "phone": "string | null",
  "email": "string | null",
  "isActive": "boolean | null"
}
```

---

### Corporation

Represents an authorized distributor in the manufacturer's network.

```json
{
  "corporationId": "string",
  "corporationName": "string",
  "extendedAttributes": [ /* DataAttribute[] */ ]
}
```

---

### CorporationCustomer

A customer account under a distributor (corporation).

```json
{
  "companyId": "string",
  "companyName": "string",
  "extendedAttributes": [ /* DataAttribute[] */ ]
}
```

---

## Claims & Labor Types

### LaborRateSchedule

Applicable labor and travel reimbursement rates for warranty work. All rate fields use `extendedAttributes` for maximum flexibility across different manufacturer programs.

```json
{
  "effectiveDate": "date | null",
  "expirationDate": "date | null",
  "extendedAttributes": [ /* DataAttribute[] */ ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `effectiveDate` | date | No | When this rate schedule takes effect |
| `expirationDate` | date | No | When this rate schedule expires |
| `extendedAttributes` | DataAttribute[] | No | All rate data as name-value pairs |

#### Example: Standard Labor Rate Schedule

```json
{
  "effectiveDate": "2025-01-01",
  "expirationDate": "2025-12-31",
  "extendedAttributes": [
    { "name": "InstallationType", "value": "Residential", "section": "Scope" },
    { "name": "Region", "value": "Southeast", "section": "Scope" },
    { "name": "LaborRate", "value": "85.00", "section": "Labor" },
    { "name": "LaborRateUOM", "value": "per_hour", "section": "Labor" },
    { "name": "MaxLaborHours", "value": "2.0", "section": "Labor" },
    { "name": "TravelRatePerMile", "value": "0.67", "section": "Travel" },
    { "name": "MaxTravelMiles", "value": "50", "section": "Travel" },
    { "name": "RefrigerantAllowance", "value": "75.00", "section": "Allowances" }
  ]
}
```

> **Note for manufacturers:** Labor rate structures vary widely between manufacturers and programs. Some use flat rates, some use hourly rates with caps, some include travel and refrigerant allowances, others don't. The `extendedAttributes` approach accommodates any structure without requiring a fixed schema.

---

## Pagination

### PaginatedResponse

Standard pagination wrapper for list endpoints.

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

## Standard Error Response

All errors follow this structure:

```json
{
  "error": "string",
  "message": "string",
  "details": {}
}
```

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `INVALID_REQUEST` | Malformed or missing required fields |
| 401 | `UNAUTHORIZED` | Missing or invalid/expired OAuth2 token |
| 403 | `FORBIDDEN` | Valid token but insufficient permissions |
| 404 | `NOT_FOUND` | Resource does not exist |
| 409 | `CONFLICT` | Duplicate or conflicting operation |
| 422 | `VALIDATION_ERROR` | Request is well-formed but fails business validation |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server-side failure |
| 503 | `SERVICE_UNAVAILABLE` | Temporary unavailability (e.g., backend system down) |

---

*End of shared data types.*
