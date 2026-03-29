# Manufacturer Warranty Hub — Endpoint Specifications

**Version:** 1.2-draft  
**Last Updated:** March 29, 2026  
**Authentication:** OAuth2 Bearer Token (see Section 1)  
**Base URL:** `https://{manufacturer-host}/v1`  

These are the endpoints the manufacturer must implement. Continuum calls these endpoints to retrieve data from and push updates to the manufacturer's systems of record.

> **Scope note:** This document covers only the manufacturer-facing endpoints — the API surface YOU build. Continuum-managed endpoints (claims CRUD, claim status, attachments, claim types, etc.) are not part of this specification.

> **ERP-agnostic:** All endpoints define a contract that the manufacturer implements as middleware. The underlying source system (SAP, Oracle, NetSuite, Epicor, custom database, etc.) is irrelevant to the contract. Your middleware translates between your internal systems and these JSON schemas.

---

## 1. Authentication

### `POST /v1/oauth2/token`

**Purpose:** Exchange client credentials for an OAuth2 access token. Continuum authenticates before making any API calls.

**Content-Type:** `application/json`

#### Request Body

```json
{
  "grant_type": "client_credentials",
  "client_id": "continuum-warranty-hub",
  "client_secret": "••••••••••••••••",
  "scope": "warranty:read warranty:write"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `grant_type` | string | **Yes** | Must be `"client_credentials"` |
| `client_id` | string | **Yes** | The client identifier issued to Continuum |
| `client_secret` | string | **Yes** | The client secret |
| `scope` | string | No | Requested scope(s), space-delimited |

#### Response `200 OK`

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "warranty:read warranty:write"
}
```

**Usage:** Include the token in all subsequent requests:
```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## 2. Unit & Product Data

### 2.1 `GET /v1/validate-unit`

**Purpose:** Validate that a serial number exists in the manufacturer's system and return the associated model number(s). This is the first call in any workflow — a lightweight check before requesting full details.

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `SerialNumber` | string | **Yes** | The serial number to validate |
| `ModelNumber` | string | No | Optional model number to narrow results (for multi-component systems) |

#### Response `200 OK`

```json
{
  "serialNumber": "ABC123456789",
  "modelNumbers": [
    {
      "modelNumber": "EN6-50-DORT",
      "modelDescription": "50gal Tall EL 4.5kW 240V",
      "installDate": "2023-06-15T00:00:00Z"
    }
  ]
}
```

#### Response `404 Not Found`

```json
{
  "error": "UNIT_NOT_FOUND",
  "message": "No unit found for serial number ABC123456789"
}
```

---

### 2.2 `GET /v1/units/{serial_number}`

**Purpose:** Return the full unit record including product details, sales transaction data, manufacturing info, installation data, registration status, and warranty summary. Powers the Warranty Verification page and auto-fill during Product Registration.

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `serial_number` | string | **Yes** | The unit's serial number |

#### Response `200 OK`

```json
{
  "serialNumber": "ABC123456789",
  "product": {
    "baseModel": "EN6-50-DORT",
    "modelDescription": "50gal Tall EL 4.5kW 2x4.5/4.5-CU/INC 240V-1ph 60Hz 2-WI AL-1A 150PSI",
    "sku": "100234567",
    "materialNumber": "200234567",
    "materialDescription": "EN6-50-DORT 200",
    "brand": "Rheem",
    "productLine": "Performance Platinum",
    "seriesName": "EN6",
    "productCategory": "Water Heater",
    "productType": "Tank",
    "fuelType": "Electric",
    "extendedAttributes": [
      { "name": "CapacityGallons", "value": "50", "section": "Capacity" },
      { "name": "Voltage", "value": "240V", "section": "Electrical" },
      { "name": "Phase", "value": "1", "section": "Electrical" },
      { "name": "Hertz", "value": "60Hz", "section": "Electrical" },
      { "name": "UEF", "value": "0.93", "section": "Efficiency" },
      { "name": "FirstHourRating", "value": "67 gal", "section": "Performance" },
      { "name": "RecoveryRate", "value": "21 GPH", "section": "Performance" }
    ]
  },
  "manufacturing": {
    "plantCode": "1600",
    "plantName": "Montgomery Plant",
    "manufactureDate": "2023-03-15",
    "lotNumber": "L2023-0315-A",
    "countryOfOrigin": "US"
  },
  "sales": {
    "salesOrg": "1030",
    "salesOrderNumber": "1234567890",
    "originalPoNumber": "42",
    "shippingPlant": "1600",
    "customerId": "5678901",
    "customerName": "Premier Plumbing Supply",
    "shipDate": "2023-03-20",
    "invoiceNumber": "INV-2023-456789",
    "invoiceDate": "2023-03-20"
  },
  "warranty": {
    "overallStatus": "active",
    "effectiveDate": "2023-06-15",
    "isTransferable": false,
    "currentOwnerIsOriginal": true
  },
  "installation": {
    "installDate": "2023-06-15",
    "installationType": "Residential",
    "installedBy": {
      "id": "CONTR-042",
      "companyName": "Smith Plumbing LLC",
      "firstName": "John",
      "lastName": "Smith"
    },
    "installationAddress": {
      "address1": "123 Main St",
      "city": "Nashville",
      "stateCode": "TN",
      "zip": "37201",
      "countryCode": "US"
    },
    "isProperlyDocumented": true
  },
  "registration": {
    "isRegistered": true,
    "registrationDate": "2023-06-20T14:30:00Z",
    "registrationSource": "web_portal",
    "registeredOwner": {
      "firstName": "Jane",
      "lastName": "Doe",
      "email": "jane.doe@email.com",
      "phone": "555-987-6543"
    },
    "affectsWarrantyTerms": false,
    "extendedCoverageDays": null
  },
  "extendedAttributes": [
    { "name": "ERPMaterialGroup", "value": "WH-ELEC-TANK", "section": "Classification" }
  ]
}
```

---

### 2.3 `GET /v1/entitlement-information`

**Purpose:** Retrieve detailed warranty entitlement information for a serial number, including per-tier coverage details, homeowner information, and purchaser details. This is the primary entitlement check used before filing a warranty claim.

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `SerialNumber` | string | **Yes** | The unit's serial number |
| `ModelNumber` | string | No | Model number (required if serial maps to multiple models) |
| `RegisteredOwnerLastName` | string | No | Last name of registered owner (for ownership verification) |
| `StateCode` | string | No | State code of installation (may affect coverage terms) |
| `CustomerId` | string | No | Distributor/customer account ID |
| `InstallationType` | string | No | `"Residential"`, `"Commercial"`, `"MultiFamily"` |

#### Response `200 OK`

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
  "coverages": [
    {
      "coverageType": "tank",
      "coverageLabel": "Tank & Heat Exchanger",
      "status": "active",
      "effectiveDate": "2023-06-15",
      "expirationDate": "2035-06-15",
      "coverageTermMonths": 144,
      "isProRated": true,
      "proRatedSchedule": {
        "fullCoverageMonths": 72,
        "tiers": [
          { "fromMonth": 1, "toMonth": 72, "reimbursementPercent": 100.0, "description": "Full replacement" },
          { "fromMonth": 73, "toMonth": 108, "reimbursementPercent": 50.0, "description": "50% credit" },
          { "fromMonth": 109, "toMonth": 144, "reimbursementPercent": 25.0, "description": "25% credit" }
        ]
      }
    },
    {
      "coverageType": "parts",
      "coverageLabel": "Component Parts",
      "status": "active",
      "effectiveDate": "2023-06-15",
      "expirationDate": "2035-06-15",
      "coverageTermMonths": 144,
      "isProRated": false
    },
    {
      "coverageType": "labor",
      "coverageLabel": "Labor",
      "status": "expired",
      "effectiveDate": "2023-06-15",
      "expirationDate": "2024-06-15",
      "coverageTermMonths": 12,
      "isProRated": false
    }
  ],
  "laborRates": {
    "effectiveDate": "2025-01-01",
    "expirationDate": "2025-12-31",
    "extendedAttributes": [
      { "name": "LaborRate", "value": "85.00", "section": "Labor" },
      { "name": "LaborRateUOM", "value": "per_hour", "section": "Labor" },
      { "name": "MaxLaborHours", "value": "2.0", "section": "Labor" },
      { "name": "TravelRatePerMile", "value": "0.67", "section": "Travel" },
      { "name": "MaxTravelMiles", "value": "50", "section": "Travel" },
      { "name": "RefrigerantAllowance", "value": "75.00", "section": "Allowances" }
    ]
  },
  "claimSubmissionDeadlineDays": 30,
  "homeowner": {
    "firstName": "Jane",
    "lastName": "Doe",
    "address": {
      "address1": "123 Main St",
      "city": "Nashville",
      "stateCode": "TN",
      "zip": "37201",
      "countryCode": "US"
    },
    "email": "jane.doe@email.com",
    "phone": "555-987-6543"
  },
  "purchaserDetails": {
    "distributorName": "Premier Plumbing Supply",
    "address": {
      "address1": "500 Commerce Blvd",
      "city": "Nashville",
      "stateCode": "TN",
      "zip": "37210"
    }
  },
  "extendedAttributes": []
}
```

---

### 2.4 `GET /v1/units/{serial_number}/warranty-coverages`

**Purpose:** Retrieve just the warranty coverage tiers for a unit. A lightweight alternative to the full entitlement endpoint when only coverage data is needed.

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `serial_number` | string | **Yes** | The unit's serial number |

#### Response `200 OK`

```json
{
  "serialNumber": "ABC123456789",
  "overallStatus": "partial",
  "effectiveDate": "2023-06-15",
  "coverages": [
    {
      "coverageType": "tank",
      "coverageLabel": "Tank & Heat Exchanger",
      "status": "active",
      "effectiveDate": "2023-06-15",
      "expirationDate": "2035-06-15",
      "coverageTermMonths": 144,
      "isProRated": true
    },
    {
      "coverageType": "parts",
      "coverageLabel": "Component Parts",
      "status": "active",
      "effectiveDate": "2023-06-15",
      "expirationDate": "2035-06-15",
      "coverageTermMonths": 144,
      "isProRated": false
    },
    {
      "coverageType": "labor",
      "coverageLabel": "Labor",
      "status": "expired",
      "effectiveDate": "2023-06-15",
      "expirationDate": "2024-06-15",
      "coverageTermMonths": 12,
      "isProRated": false
    }
  ],
  "extendedAttributes": []
}
```

---

### 2.5 `GET /v1/units/{serial_number}/parts`

**Purpose:** Return the bill of materials (BOM) for a unit — the list of serviceable replacement parts with images and specifications. Powers the "Replacement Parts" section of Warranty Verification and the parts selection step of claim filing.

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `serial_number` | string | **Yes** | The unit's serial number |

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `include_images` | boolean | No | Include inline image URLs in response (default: `true`) |
| `in_stock_only` | boolean | No | Filter to only parts currently in stock |
| `serviceable_only` | boolean | No | Filter to only individually serviceable/replaceable parts (default: `true`) |

#### Response `200 OK`

```json
{
  "serialNumber": "ABC123456789",
  "baseModel": "EN6-50-DORT",
  "parts": [
    {
      "partNumber": "SP20042",
      "description": "K,ANODE,29\",3/4\"NPT,.75\"DIA,ALUMINUM",
      "crossReferences": [
        { "referenceType": "supersedes", "partNumber": "SP20041", "description": "Previous version — lower corrosion resistance" },
        { "referenceType": "brand_xref", "partNumber": "SP20042-RU", "brand": "Ruud" },
        { "referenceType": "upc", "partNumber": "020352330426" }
      ],
      "quantity": 1,
      "unitOfMeasure": "EA",
      "partType": "anode",
      "partSubType": null,
      "brand": "Rheem",
      "isServiceable": true,
      "isWarrantyCovered": true,
      "warrantyExpirationDate": "2035-06-15",
      "inStock": true,
      "images": [
        {
          "id": "img-sp20042-01",
          "mediaType": "image",
          "category": "part_photo",
          "url": "https://parts.example.com/images/SP20042_main.jpg",
          "width": 800,
          "height": 800
        }
      ],
      "notes": null,
      "replacementsAvailable": 2,
      "extendedAttributes": [
        { "name": "Dimensions", "value": "29.0\" × 0.75\" × 0.75\"", "section": "Physical" },
        { "name": "Weight", "value": "1.25 lbs", "section": "Physical" },
        { "name": "MaterialType", "value": "Aluminum", "section": "Physical" },
        { "name": "ThreadSize", "value": "3/4\" NPT", "section": "Physical" },
        { "name": "Compatibility", "value": "EN6-50-DORT series", "section": "Fitment" },
        { "name": "InstallationNotes", "value": "Requires 1-1/16\" socket.", "section": "Service" }
      ]
    },
    {
      "partNumber": "SP10869GH",
      "description": "ELEMENT,SCREW-IN,4500W,240V,COPPER,HIGH DENSITY",
      "crossReferences": [
        { "referenceType": "upc", "partNumber": "020352330495" },
        { "referenceType": "brand_xref", "partNumber": "SP10869GH-RU", "brand": "Ruud" }
      ],
      "quantity": 2,
      "unitOfMeasure": "EA",
      "partType": "element",
      "partSubType": "heating_element",
      "brand": "Rheem",
      "isServiceable": true,
      "isWarrantyCovered": true,
      "warrantyExpirationDate": "2035-06-15",
      "inStock": true,
      "images": [],
      "notes": "Upper and lower elements are identical on this model.",
      "replacementsAvailable": 3,
      "extendedAttributes": [
        { "name": "Voltage", "value": "240V", "section": "Electrical" },
        { "name": "Wattage", "value": "4500W", "section": "Electrical" },
        { "name": "MaterialType", "value": "Copper", "section": "Physical" },
        { "name": "InstallationNotes", "value": "Use element wrench. Drain tank before replacing.", "section": "Service" }
      ]
    }
  ],
  "totalParts": 8
}
```

---

### 2.6 `GET /v1/units/{serial_number}/distributor`

**Purpose:** Return the distributor that originally purchased/sold the unit. Used to auto-fill distributor information during Product Registration.

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `serial_number` | string | **Yes** | The unit's serial number |

#### Response `200 OK`

```json
{
  "distributorId": "DIST-001",
  "name": "Premier Plumbing Supply",
  "accountNumber": "5678901",
  "purchaseDate": "2023-03-18",
  "address": {
    "address1": "500 Commerce Blvd",
    "city": "Nashville",
    "stateCode": "TN",
    "zip": "37210",
    "countryCode": "US"
  },
  "phone": "615-555-0100",
  "email": "orders@premierplumbing.com"
}
```

---

## 3. Parts Lookup & Cross-Reference

### 3.1 `GET /v1/bill-of-materials`

**Purpose:** Retrieve the BOM for a serial/model in the context of warranty claims. Returns a simplified parts list for the claims parts-selection step.

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `SerialNumber` | string | **Yes** | The unit's serial number |
| `ModelNumber` | string | No | Model number (required if serial maps to multiple models) |

#### Response `200 OK`

```json
{
  "serialNumber": "ABC123456789",
  "modelNumber": "EN6-50-DORT",
  "parts": [
    {
      "partNumber": "SP20042",
      "description": "K,ANODE,29\",3/4\"NPT,.75\"DIA,ALUMINUM",
      "partType": "anode",
      "crossReferences": [
        { "referenceType": "supersedes", "partNumber": "SP20041" }
      ]
    },
    {
      "partNumber": "SP10869GH",
      "description": "ELEMENT,SCREW-IN,4500W,240V,COPPER,HIGH DENSITY",
      "partType": "element",
      "crossReferences": []
    },
    {
      "partNumber": "SP8346",
      "description": "THERMOSTAT,UPPER,SNAP-DISC,170F",
      "partType": "thermostat",
      "crossReferences": []
    }
  ]
}
```

---

### 3.2 `GET /v1/parts/cross-reference`

**Purpose:** Look up a part's cross-references by part number. Since this is a manufacturer's API, cross-references primarily show **supersession chains** (what this part replaces or is replaced by) and **brand-specific equivalents** (the same part under the manufacturer's different brand names).

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `partNumber` | string | **Yes** | The part number to look up cross-references for |

#### Response `200 OK`

```json
{
  "partNumber": "SP20042",
  "description": "K,ANODE,29\",3/4\"NPT,.75\"DIA,ALUMINUM",
  "brand": "Rheem",
  "crossReferences": [
    {
      "referenceType": "supersedes",
      "partNumber": "SP20041",
      "description": "Previous version — replaced due to improved corrosion resistance"
    },
    {
      "referenceType": "superseded_by",
      "partNumber": "SP20042-V2",
      "description": "Newer version available with extended lifespan coating"
    },
    {
      "referenceType": "brand_xref",
      "partNumber": "SP20042-RU",
      "brand": "Ruud",
      "description": "Same part under the Ruud brand"
    },
    {
      "referenceType": "brand_xref",
      "partNumber": "SP20042-RC",
      "brand": "Richmond",
      "description": "Same part under the Richmond brand"
    },
    {
      "referenceType": "oem_xref",
      "partNumber": "A-29-075-AL",
      "brand": "Cam Manufacturing",
      "description": "Component manufacturer's part number"
    },
    {
      "referenceType": "upc",
      "partNumber": "020352330426"
    }
  ],
  "extendedAttributes": []
}
```

**Cross-reference types for manufacturers:**

| Type | Description | Example |
|------|-------------|---------|
| `supersedes` | This part replaces an older part | SP20042 supersedes SP20041 |
| `superseded_by` | This part has been replaced by a newer part | SP20042 superseded by SP20042-V2 |
| `brand_xref` | Same part sold under a different brand name owned by the manufacturer | Rheem SP20042 = Ruud SP20042-RU |
| `oem_xref` | The component supplier's own part number | Rheem SP20042 = Cam Mfg A-29-075-AL |
| `upc` | Universal Product Code / barcode | 020352330426 |
| `equivalent` | Functionally equivalent part (different design, same fit/form/function) | — |

---

### 3.3 `GET /v1/products/warranty-codes`

**Purpose:** Retrieve the hierarchical warranty disposition codes for a given part. These are the codes used to classify why a part failed, organized in a tree structure (general → specific → detail).

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `SerialNumber` | string | No | Serial number (may affect available codes) |
| `ModelNumber` | string | No | Model number |
| `PartNumber` | string | No | The specific part to get codes for |

#### Response `200 OK`

```json
{
  "serialNumberRequired": false,
  "reasonCodes": [
    {
      "id": "ELEC",
      "description": "Electrical Failure",
      "fullPath": "Electrical Failure",
      "idPath": "ELEC",
      "children": [
        {
          "id": "THERM",
          "description": "Thermostat",
          "fullPath": "Electrical Failure > Thermostat",
          "idPath": "ELEC/THERM",
          "children": [
            {
              "id": "FAIL-REG",
              "description": "Failed to regulate temperature",
              "fullPath": "Electrical Failure > Thermostat > Failed to regulate temperature",
              "idPath": "ELEC/THERM/FAIL-REG",
              "requiresAttachment": false,
              "children": []
            }
          ]
        }
      ]
    },
    {
      "id": "LEAK",
      "description": "Leak / Water Damage",
      "fullPath": "Leak / Water Damage",
      "idPath": "LEAK",
      "children": [
        {
          "id": "TANK-LEAK",
          "description": "Tank leak",
          "fullPath": "Leak / Water Damage > Tank leak",
          "idPath": "LEAK/TANK-LEAK",
          "requiresAttachment": true,
          "children": []
        }
      ]
    }
  ]
}
```

---

### 3.4 `GET /v1/products/replacements`

**Purpose:** Retrieve available replacement parts for a failed part. Returns the original part (if still available), any superseding parts, and optionally non-OEM alternatives.

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `PartNumber` | string | **Yes** | The failed part number |
| `IncludeNonOEM` | boolean | No | Include non-OEM replacement options (default: `false`) |

#### Response `200 OK`

```json
{
  "replacementParts": [
    {
      "partNumber": "SP20042",
      "description": "K,ANODE,29\",3/4\"NPT,.75\"DIA,ALUMINUM",
      "crossReferences": [
        { "referenceType": "upc", "partNumber": "020352330426" }
      ],
      "brand": "Rheem",
      "isOEM": true,
      "replacementType": "exact",
      "inStock": true,
      "listPrice": 45.99,
      "warrantyPrice": 0.00,
      "score": 10,
      "scoreDetail": "Exact match — same part number",
      "images": [
        {
          "id": "img-sp20042-01",
          "mediaType": "image",
          "category": "part_photo",
          "url": "https://parts.example.com/images/SP20042_main.jpg"
        }
      ],
      "extendedAttributes": []
    },
    {
      "partNumber": "SP20042-V2",
      "description": "K,ANODE,29\",3/4\"NPT,.75\"DIA,ALUMINUM (IMPROVED)",
      "crossReferences": [
        { "referenceType": "supersedes", "partNumber": "SP20042", "description": "Improved corrosion resistance" }
      ],
      "brand": "Rheem",
      "isOEM": true,
      "replacementType": "superseded",
      "inStock": true,
      "listPrice": 52.99,
      "warrantyPrice": 0.00,
      "score": 9,
      "scoreDetail": "Manufacturer-recommended superseding replacement",
      "images": [],
      "extendedAttributes": []
    }
  ]
}
```

---

## 4. Media

### 4.1 `GET /v1/media`

**Purpose:** Retrieve media (documents, images, manuals, diagrams, etc.) associated with a model or part. This is a unified endpoint — all media types are accessible here. Query by model number to get product documentation, or by part number to get part-specific media.

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `model_number` | string | **Yes*** | The product model number. At least one of `model_number` or `part_number` is required. |
| `part_number` | string | **Yes*** | The part number. At least one of `model_number` or `part_number` is required. |
| `mediaType` | string | No | Filter by media type: `"document"`, `"image"`, `"video"`, `"diagram"` |
| `category` | string | No | Filter by category (e.g., `"installation_guide"`, `"wiring_diagram"`, `"part_photo"`) |

#### Response `200 OK` — Query by model number

`GET /v1/media?model_number=EN6-50-DORT`

```json
{
  "media": [
    {
      "id": "doc-001",
      "title": "Installation Guide",
      "filename": "EN6-50-DORT_Installation_Guide.pdf",
      "mediaType": "document",
      "category": "installation_guide",
      "url": "https://docs.example.com/EN6-50-DORT_Installation_Guide.pdf",
      "previewUrl": "https://docs.example.com/previews/doc-001-thumb.jpg",
      "mimeType": "application/pdf",
      "fileSizeBytes": 2621440,
      "pageCount": 24,
      "lastUpdated": "2024-06-01",
      "language": "en",
      "associatedModels": ["EN6-50-DORT", "EN6-40-DORT"],
      "associatedPartNumbers": [],
      "extendedAttributes": []
    },
    {
      "id": "doc-002",
      "title": "Wiring Diagram",
      "filename": "EN6-50-DORT_Wiring_Diagram.pdf",
      "mediaType": "document",
      "category": "wiring_diagram",
      "url": "https://docs.example.com/EN6-50-DORT_Wiring_Diagram.pdf",
      "previewUrl": null,
      "mimeType": "application/pdf",
      "fileSizeBytes": 524288,
      "pageCount": 2,
      "lastUpdated": "2024-03-15",
      "language": "en",
      "associatedModels": ["EN6-50-DORT"],
      "associatedPartNumbers": [],
      "extendedAttributes": []
    },
    {
      "id": "img-en6-01",
      "title": "EN6-50-DORT Product Photo",
      "mediaType": "image",
      "category": "product_photo",
      "url": "https://images.example.com/EN6-50-DORT_main.jpg",
      "previewUrl": "https://images.example.com/EN6-50-DORT_thumb.jpg",
      "mimeType": "image/jpeg",
      "width": 1200,
      "height": 1200,
      "associatedModels": ["EN6-50-DORT"],
      "associatedPartNumbers": [],
      "extendedAttributes": []
    }
  ],
  "total": 3
}
```

#### Response `200 OK` — Query by part number

`GET /v1/media?part_number=SP20042`

```json
{
  "media": [
    {
      "id": "img-sp20042-01",
      "title": "Anode Rod SP20042 — Primary",
      "mediaType": "image",
      "category": "part_photo",
      "url": "https://parts.example.com/images/SP20042_main.jpg",
      "mimeType": "image/jpeg",
      "width": 800,
      "height": 800,
      "associatedModels": [],
      "associatedPartNumbers": ["SP20042"],
      "extendedAttributes": []
    },
    {
      "id": "img-sp20042-02",
      "title": "Anode Rod SP20042 — Installed View",
      "mediaType": "image",
      "category": "installation_photo",
      "url": "https://parts.example.com/images/SP20042_installed.jpg",
      "mimeType": "image/jpeg",
      "width": 1200,
      "height": 800,
      "associatedModels": [],
      "associatedPartNumbers": ["SP20042"],
      "extendedAttributes": []
    }
  ],
  "total": 2
}
```

---

### 4.2 `GET /v1/media/{id}`

**Purpose:** Retrieve a specific media item by its unique ID. Use this when you already have a media ID from a previous response and need the full details or to build a direct link.

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | **Yes** | The unique media identifier |

#### Response `200 OK`

```json
{
  "id": "doc-001",
  "title": "Installation Guide",
  "filename": "EN6-50-DORT_Installation_Guide.pdf",
  "mediaType": "document",
  "category": "installation_guide",
  "url": "https://docs.example.com/EN6-50-DORT_Installation_Guide.pdf",
  "previewUrl": "https://docs.example.com/previews/doc-001-thumb.jpg",
  "mimeType": "application/pdf",
  "fileSizeBytes": 2621440,
  "pageCount": 24,
  "lastUpdated": "2024-06-01",
  "language": "en",
  "associatedModels": ["EN6-50-DORT", "EN6-40-DORT"],
  "associatedPartNumbers": [],
  "extendedAttributes": []
}
```

---

### 4.3 `POST /v1/media` *(Manual Upload)*

**Purpose:** Upload media for a product model or part. Used when the manufacturer wants to manually add content through the platform.

**Content-Type:** `multipart/form-data`

#### Request Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | binary | **Yes** | The file to upload |
| `title` | string | **Yes** | Media title |
| `mediaType` | string | **Yes** | `"document"`, `"image"`, `"video"`, `"diagram"` |
| `category` | string | **Yes** | Category value (see enum in [Data Types](./data-types.md)) |
| `model_number` | string | No | Associated model number |
| `part_number` | string | No | Associated part number |

#### Response `201 Created`

```json
{
  "id": "media-new-001",
  "url": "https://docs.example.com/uploaded/media-new-001.pdf",
  "uploadedAt": "2024-07-20T10:00:00Z"
}
```

---

## 5. Reference Data

### 5.1 `GET /v1/contractors`

**Purpose:** Search the contractor database. Used to populate the contractor selection dropdown during Product Registration and claims filing.

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `search` | string | No | Free-text search on name/company |
| `zip_code` | string | No | Filter by service area zip code |
| `certified_only` | boolean | No | Only return contractors certified by this manufacturer |
| `page` | integer | No | Page number (default: 1) |
| `page_size` | integer | No | Results per page (default: 25) |

#### Response `200 OK`

```json
{
  "contractors": [
    {
      "id": "CONTR-042",
      "companyName": "Smith Plumbing LLC",
      "firstName": "John",
      "lastName": "Smith",
      "phone": "555-123-4567",
      "email": "john@smithplumbing.com",
      "licenseNumber": "PL-2023-45678",
      "isCertified": true,
      "certifications": ["Rheem Certified Pro", "Tankless Specialist"],
      "address": {
        "address1": "456 Trade St",
        "city": "Houston",
        "stateCode": "TX",
        "zip": "77001",
        "countryCode": "US"
      }
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 25,
    "totalRecords": 1,
    "totalPages": 1
  }
}
```

---

### 5.2 `GET /v1/locations`

**Purpose:** Retrieve registered warranty service locations (warehouses, branches, distribution centers).

#### Response `200 OK`

```json
{
  "locations": [
    {
      "locationId": "LOC-001",
      "locationName": "Nashville Distribution Center",
      "locationType": "distribution_center",
      "address": {
        "address1": "1200 Logistics Parkway",
        "city": "Nashville",
        "stateCode": "TN",
        "zip": "37210",
        "countryCode": "US"
      },
      "phone": "615-555-0200",
      "email": "nashville@manufacturer.com",
      "isActive": true
    }
  ]
}
```

---

### 5.3 `GET /v1/corporations`

**Purpose:** Retrieve the list of authorized distributors (corporations) in the manufacturer's network.

#### Response `200 OK`

```json
{
  "corporations": [
    {
      "corporationId": "CORP-001",
      "corporationName": "Premier Plumbing Supply",
      "extendedAttributes": []
    }
  ]
}
```

---

### 5.4 `GET /v1/corporations/details`

**Purpose:** Retrieve details for a specific distributor (corporation).

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `CorporationId` | string | **Yes** | The corporation's unique identifier |

#### Response `200 OK`

```json
{
  "corporationId": "CORP-001",
  "corporationName": "Premier Plumbing Supply",
  "extendedAttributes": [
    { "name": "Region", "value": "Southeast", "section": "Classification" },
    { "name": "TierLevel", "value": "Platinum", "section": "Partnership" }
  ]
}
```

---

### 5.5 `GET /v1/corporations/customers`

**Purpose:** Retrieve customer accounts under a specific distributor (corporation).

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `CorporationId` | string | **Yes** | The corporation's unique identifier |

#### Response `200 OK`

```json
{
  "customers": [
    {
      "companyId": "C-827451",
      "companyName": "Smith Plumbing LLC",
      "extendedAttributes": []
    }
  ]
}
```

---

### 5.6 `GET /v1/labor-rates`

**Purpose:** Retrieve the applicable labor and travel reimbursement rate schedule. Rate structures vary widely between manufacturers — all rate data is carried in `extendedAttributes` for maximum flexibility.

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `StateCode` | string | No | State code to determine regional rates |
| `ZipCode` | string | No | Zip code for more precise regional lookup |
| `InstallationType` | string | No | `"Residential"`, `"Commercial"`, `"MultiFamily"` |
| `ClaimType` | string | No | Claim type code (if rates vary by claim type) |

#### Response `200 OK`

```json
{
  "rates": [
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
  ]
}
```

---

## 6. Registration Sync

### 6.1 `POST /v1/registrations/sync`

**Purpose:** Push a product registration from Continuum back to the manufacturer's system. Ensures the manufacturer's warranty database reflects that the unit has been registered, which may activate or extend warranty coverage.

#### Request Body

```json
{
  "serialNumber": "ABC123456789",
  "modelNumber": "EN6-50-DORT",
  "registrationDate": "2024-01-15T14:30:00Z",
  "installDate": "2024-01-10",
  "installationType": "Residential",
  "homeowner": {
    "firstName": "Jane",
    "lastName": "Doe",
    "email": "jane.doe@email.com",
    "phone": "555-987-6543",
    "address": {
      "address1": "123 Main St",
      "city": "Nashville",
      "stateCode": "TN",
      "zip": "37201",
      "countryCode": "US"
    }
  },
  "contractor": {
    "id": "CONTR-042",
    "companyName": "Smith Plumbing LLC"
  },
  "distributor": {
    "distributorId": "DIST-001",
    "name": "Premier Plumbing Supply",
    "purchaseDate": "2023-12-20"
  },
  "purchaseDate": "2024-01-05"
}
```

#### Response `200 OK`

```json
{
  "synced": true,
  "manufacturerRegistrationId": "REG-MFR-2024-00456",
  "warrantyActivated": true,
  "coverageAdjustments": [
    {
      "coverageType": "tank",
      "previousExpirationDate": null,
      "newExpirationDate": "2036-01-10",
      "reason": "Registration activated full warranty from install date"
    }
  ],
  "message": "Registration synced successfully. Warranty activated from install date."
}
```

#### Response `409 Conflict`

```json
{
  "error": "ALREADY_REGISTERED",
  "message": "Serial number ABC123456789 is already registered.",
  "details": {
    "existingRegistrationDate": "2023-12-01T00:00:00Z",
    "registeredOwner": "John Doe"
  }
}
```

---

## 7. Historical Data Import

### 7.1 `GET /v1/import/registrations`

**Purpose:** Bulk export existing warranty registration data from the manufacturer's legacy systems. Powers the one-time (or recurring) data import so Continuum starts with the full registration history.

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `since` | ISO 8601 date | No | Only return registrations after this date (for incremental sync) |
| `page` | integer | No | Page number (default: 1) |
| `page_size` | integer | No | Results per page (default: 100, max: 1000) |

#### Response `200 OK`

```json
{
  "registrations": [
    {
      "serialNumber": "ABC123456789",
      "modelNumber": "EN6-50-DORT",
      "brand": "Rheem",
      "registrationDate": "2022-06-15T00:00:00Z",
      "purchaseDate": "2022-06-01",
      "installDate": "2022-06-10",
      "installationType": "Residential",
      "distributor": {
        "distributorId": "DIST-003",
        "name": "Pro Water Systems Inc",
        "accountNumber": "5678903"
      },
      "homeowner": {
        "firstName": "Bob",
        "lastName": "Johnson",
        "email": "bob@example.com",
        "phone": "555-111-2222",
        "address": {
          "address1": "789 Oak Ave",
          "city": "Atlanta",
          "stateCode": "GA",
          "zip": "30301",
          "countryCode": "US"
        }
      },
      "warranty": {
        "overallStatus": "active",
        "effectiveDate": "2022-06-10"
      },
      "coverages": [
        { "coverageType": "tank", "status": "active", "expirationDate": "2034-06-10", "coverageTermMonths": 144 },
        { "coverageType": "parts", "status": "active", "expirationDate": "2034-06-10", "coverageTermMonths": 144 },
        { "coverageType": "labor", "status": "expired", "expirationDate": "2023-06-10", "coverageTermMonths": 12 }
      ]
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 100,
    "totalRecords": 45230,
    "totalPages": 453
  }
}
```

---

## 8. Health Check

### 8.1 `GET /v1/health`

**Purpose:** Basic health check endpoint. Continuum will periodically call this to verify the middleware API is operational.

#### Response `200 OK`

```json
{
  "status": "Healthy",
  "timestamp": "2026-03-29T14:30:00Z",
  "version": "1.0"
}
```

---

## Endpoint Summary

### Manufacturer Must Implement (22 endpoints)

| # | Endpoint | Method | Purpose |
|---|----------|--------|---------|
| 1 | `/v1/oauth2/token` | POST | Authentication |
| 2 | `/v1/health` | GET | Health check |
| 3 | `/v1/validate-unit` | GET | Validate serial, return model(s) |
| 4 | `/v1/units/{serial_number}` | GET | Full unit record |
| 5 | `/v1/entitlement-information` | GET | Warranty entitlement + coverages |
| 6 | `/v1/units/{serial_number}/warranty-coverages` | GET | Coverage tiers only |
| 7 | `/v1/units/{serial_number}/parts` | GET | BOM with specs + images |
| 8 | `/v1/units/{serial_number}/distributor` | GET | Distributor lookup |
| 9 | `/v1/bill-of-materials` | GET | BOM for claims context |
| 10 | `/v1/parts/cross-reference` | GET | Part supersedes and brand cross-references |
| 11 | `/v1/products/warranty-codes` | GET | Hierarchical warranty codes |
| 12 | `/v1/products/replacements` | GET | Replacement part options |
| 13 | `/v1/media` | GET | Media by model or part (docs, images, manuals) |
| 14 | `/v1/media/{id}` | GET | Specific media item by ID |
| 15 | `/v1/media` | POST | Manual media upload |
| 16 | `/v1/contractors` | GET | Contractor directory |
| 17 | `/v1/locations` | GET | Warranty locations |
| 18 | `/v1/corporations` | GET | Authorized distributors |
| 19 | `/v1/corporations/details` | GET | Distributor details |
| 20 | `/v1/corporations/customers` | GET | Customers under distributor |
| 21 | `/v1/labor-rates` | GET | Labor/travel rate schedule |
| 22 | `/v1/registrations/sync` | POST | Push registration to manufacturer |
| 23 | `/v1/import/registrations` | GET | Bulk historical registration export |

### Continuum-Managed (NOT in Manufacturer Documentation)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/claim-types` | GET | Available claim types |
| `/v1/claim/validate` | POST | Pre-validate a claim |
| `/v1/claim` | POST | Create warranty claim |
| `/v1/claim` | PATCH | Update warranty claim |
| `/v1/claim` | DELETE | Delete warranty claim |
| `/v1/claim/status` | GET | Check claim status |
| `/v1/claim/upload-attachment` | POST | Upload claim attachments |
| `/v1/claim/{claimNumber}/return-instructions` | GET | Return shipping details |
| `/v1/claim/{claimNumber}/return-confirmation` | POST | Confirm return receipt |
| `/v1/units/{serial_number}/claim-history` | GET | Past claims for a serial |
| Webhook subscription | POST | Register callback URL |

---

*End of endpoint specifications.*
