# 🛒 Order Management System — Big Billion Day
> Built with MuleSoft API-Led Connectivity | Experience · Process · System APIs

![MuleSoft](https://img.shields.io/badge/MuleSoft-Anypoint_Platform-00A0DF?style=flat&logo=mulesoft)
![MySQL](https://img.shields.io/badge/Database-MySQL_8.0-4479A1?style=flat&logo=mysql&logoColor=white)
![CloudHub](https://img.shields.io/badge/Deployed_on-CloudHub_2.0-00A0DF?style=flat)
![RAML](https://img.shields.io/badge/API_Spec-RAML_1.0-FF6B35?style=flat)

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Database Setup](#-database-setup)
- [RAML Design & API Specification](#-raml-design--api-specification)
- [API Fragments & Exchange Dependencies](#-api-fragments--exchange-dependencies)
- [Mule Implementation](#-mule-implementation)
- [Properties & Secure Configuration](#-properties--secure-configuration)
- [Global Error Handling](#-global-error-handling)
- [Local Setup & Running](#-local-setup--running)
- [Postman Testing](#-postman-testing)
- [CloudHub Deployment](#-cloudhub-deployment)
- [Applying Policies — Client ID Enforcement](#-applying-policies--client-id-enforcement)
- [Assumptions & Limitations](#-assumptions--limitations)

---

## 📌 Project Overview

The **Order Management System (OMS)** is a MuleSoft-based API project built for a high-traffic sale event — **Big Billion Day**. It allows customers to place bulk orders in a single request and returns per-item order outcomes based on real-time inventory availability.

### Business Rules

| Rule | Detail |
|---|---|
| Unique customer key | `phoneNumber` (10-digit numeric) |
| Stock available | `status: "Order Placed"` + `orderId` generated |
| Stock unavailable | `status: "Sorry, not placed"` |
| Stock decrement | Immediate and atomic on successful placement |
| Request must NOT contain | `orderId` (generated internally by Process API) |

### Sample Response

```json
{
  "phoneNumber": "9876543210",
  "requestStatus": "PARTIAL_SUCCESS",
  "itemResults": [
    { "sku": "MOB-IPH15-128-BLK", "quantity": 1, "status": "Order Placed", "orderId": "ORD-20261012-000091" },
    { "sku": "LAP-DELL-5480",     "quantity": 2, "status": "Sorry, not placed" },
    { "sku": "EAR-BT-SONY-WH",   "quantity": 1, "status": "Order Placed", "orderId": "ORD-20261012-000092" }
  ]
}
```

---

## 🏗 Architecture

This project follows **MuleSoft API-Led Connectivity** with 3 distinct layers:

```
Customer / Postman
        │
        │  POST /orders
        │  POST /stock/availability
        │  Headers: client_id, client_secret
        ▼
┌────────────────────────────────────────┐
│         EXPERIENCE API  (EAPI)         │
│  Port : 8081 (local) / CloudHub (80)   │
│  Path : experience/*                   │
│  Role : Customer-facing, thin layer    │
│  Auth : Client ID Enforcement Policy   │
└────────────────────────────────────────┘
        │  x-correlation-id forwarded
        │  No credentials passed downstream
        ▼
┌────────────────────────────────────────┐
│         PROCESS API  (PAPI)            │
│  Port : 8082 (local) / CloudHub (80)   │
│  Path : process/*                      │
│  Role : Business logic, orchestration  │
│  Auth : None (internal only)           │
└────────────────────────────────────────┘
        │  Calls System API 3 times:
        │  1. POST /customers  (upsert)
        │  2. POST /stock/availability (bulk check)
        │  3. POST /orders (once per eligible item)
        ▼
┌────────────────────────────────────────┐
│         SYSTEM API  (SAPI)             │
│  Port : 8083 (local) / CloudHub (80)   │
│  Path : system/*                       │
│  Role : DB operations only             │
│  Auth : None (internal only)           │
└────────────────────────────────────────┘
        │
        ▼
   MySQL Database
   (Hosted on filess.io — free shared server)
```

### Why API-Led Connectivity?

Each layer is **independently deployable** and has a **single responsibility**:

- **EAPI** — Exposes public-facing endpoints. Security policy applied here only.
- **PAPI** — All business logic lives here. Validates input, orchestrates calls, generates `orderId`, shapes response.
- **SAPI** — Talks to MySQL only. No business logic. CRUD operations only.

> If the database changes from MySQL to Oracle — only SAPI changes. EAPI and PAPI are untouched.

---

## 🛠 Tech Stack

| Component | Technology |
|---|---|
| Integration Platform | MuleSoft Anypoint Studio 7.x |
| Runtime | Mule 4.6.x |
| API Specification | RAML 1.0 |
| API Design | Anypoint Design Center |
| API Exchange | Anypoint Exchange |
| Database | MySQL 8.0 (hosted on filess.io) |
| Deployment | CloudHub 2.0 |
| API Management | Anypoint API Manager |
| Security | Secure Properties + Client ID Enforcement Policy |
| Testing | Postman |

---

## 🗄 Database Setup

### Why filess.io?

CloudHub-deployed Mule applications cannot access a locally hosted (`localhost`) database because CloudHub runs in Salesforce's cloud infrastructure. **filess.io** provides a free shared MySQL server with a public hostname, making it accessible from CloudHub.

### Connection Details Structure

```
Host     : cxlt01.h.filess.io   (public hostname)
Port     : 3307
Database : omsDB_pleasurebe
User     : omsDB_pleasurebe
```

> ⚠️ Actual credentials are stored in encrypted properties files and never committed to source control.

### Schema

```sql
CREATE DATABASE omsdb;
USE omsdb;

-- Stores customer details
-- phone_number is the unique customer key
CREATE TABLE customers (
  id            INT AUTO_INCREMENT PRIMARY KEY,
  full_name     VARCHAR(100)        NOT NULL,
  phone_number  VARCHAR(10) UNIQUE,
  email         VARCHAR(100)        NOT NULL,
  address_line1 VARCHAR(200)        NOT NULL,
  address_line2 VARCHAR(200),
  city          VARCHAR(100)        NOT NULL,
  state         VARCHAR(10)         NOT NULL,
  postal_code   VARCHAR(10)         NOT NULL
);

-- Source of truth for product stock
CREATE TABLE inventory (
  id                 INT AUTO_INCREMENT PRIMARY KEY,
  sku                VARCHAR(100) UNIQUE NOT NULL,
  available_quantity INT DEFAULT 0       NOT NULL
);

-- One record per successfully placed item
-- order_id is UNIQUE — prevents duplicate orders
CREATE TABLE orders (
  id             INT AUTO_INCREMENT PRIMARY KEY,
  order_id       VARCHAR(50) UNIQUE  NOT NULL,
  phone_number   VARCHAR(10)         NOT NULL,
  sku            VARCHAR(100)        NOT NULL,
  quantity       INT                 NOT NULL,
  order_channel  VARCHAR(50),
  payment_mode   VARCHAR(50),
  delivery_pref  VARCHAR(50),
  status         VARCHAR(30)         NOT NULL,
  FOREIGN KEY (phone_number) REFERENCES customers(phone_number)
);
```

### Sample Inventory Data

```sql
INSERT INTO inventory (sku, available_quantity) VALUES
('MOB-IPH15-128-BLK', 14),
('LAP-DELL-5480',      0),
('EAR-BT-SONY-WH',    38),
('TV-SAM-55-UHD',      0),
('TAB-SAM-S9',         7),
('MOB-SAM-S24-WHT',   25),
('LAP-HP-PAVILION',   12);
```

> Full schema with data is available in [`database/output_file.sql`](database/output_file.sql)

---

## 📐 RAML Design & API Specification

All 3 API specifications were designed in **Anypoint Design Center** using **RAML 1.0**.

### Project Structure (per API)

```
api-name/
├── api-name.raml                  ← Root specification
├── examples/
│   ├── requests/
│   │   ├── orderPostRequestExample.json
│   │   └── stockCheckPostRequestExample.json
│   └── responses/
│       ├── orderPostResponseExample.json
│       └── stockCheckPostResponseExample.json
└── exchange_modules/              ← Reusable fragments from Exchange
    └── {groupId}/
        ├── common-data-types/
        ├── common-error-responses/
        ├── resourcetypes-library/
        └── security-schemes-oms/  ← EAPI only
```

### API Endpoints Summary

| API | Endpoint | Method | Description |
|---|---|---|---|
| EAPI | `/orders` | POST | Place bulk orders (customer-facing) |
| EAPI | `/stock/availability` | POST | Check stock for SKUs (customer-facing) |
| PAPI | `/orders` | POST | Orchestrate order placement (internal) |
| PAPI | `/stock/availability` | POST | Delegate stock check (internal) |
| SAPI | `/orders` | POST | Insert order + decrement stock (DB) |
| SAPI | `/customers` | POST | Upsert customer record (DB) |
| SAPI | `/stock/availability` | POST | Query inventory (DB) |

---

## 🧩 API Fragments & Exchange Dependencies

Reusable RAML components were built as **Exchange fragments** and shared across all 3 APIs. This avoids duplication and ensures consistency.

### Fragment 1 — common-data-types (Library)

**Published as:** `RAML 1.0 Library`

Contains all shared data types used across all 3 APIs:

| Type | Used In | Purpose |
|---|---|---|
| `Address` | EAPI, PAPI | Customer address object |
| `Customer` | EAPI, PAPI, SAPI | Customer details |
| `Item` | EAPI, PAPI | Order item (sku + quantity) |
| `Order` | EAPI, PAPI | Full order request |
| `OrderResponse` | EAPI, PAPI | Order result response |
| `ItemResult` | EAPI, PAPI | Per-item outcome |
| `StockRequest` | All 3 | SKU list for stock check |
| `StockItem` | All 3 | Per-SKU stock result |
| `StockResponse` | All 3 | Full stock check response |
| `OrderPlaceRequest` | SAPI | Single item order to DB |
| `OrderPlaceResponse` | SAPI | DB operation result |
| `CustomerResponse` | SAPI | Upsert confirmation |

**How to use in RAML:**
```raml
uses:
  common: exchange_modules/{groupId}/common-data-types/1.0.3/common-data-types.raml

types:
  orderPostRequestDataType: common.Order
  stockCheckRequestDataType: common.StockRequest
```

### Fragment 2 — common-error-responses (Trait)

**Published as:** `RAML 1.0 Trait`

Provides standard error responses (400, 404, 500) applied to every endpoint via `is: [errorResponses]`.

```raml
traits:
  errorResponses: !include
    exchange_modules/{groupId}/common-error-responses/1.0.2/common-error-responses.raml
```

All error responses follow this structure:
```json
{
  "errorCode":     "BAD_REQUEST",
  "message":       "Human readable description",
  "correlationId": "uuid-correlation-id",
  "timestamp":     "2026-04-22T06:00:00Z"
}
```

### Fragment 3 — resourcetypes-library (ResourceType)

**Published as:** `RAML 1.0 ResourceType`

A parameterized POST resource type that accepts description, request/response types and examples. Reduces RAML boilerplate significantly.

```raml
resourceTypes:
  resourceType-post: !include
    exchange_modules/{groupId}/resourcetypes-library/1.0.0/resourceType-post.raml

/orders:
  type:
    resourceType-post:
      description:         "Place bulk orders"
      requestType:         orderPostRequestDataType
      requestExample:      !include examples/requests/orderPostRequestExample.json
      responseType:        orderPostResponseDataType
      responseExample:     !include examples/responses/orderPostResponseExample.json
      responseDescription: "Order processing complete"
```

### Fragment 4 — security-schemes-oms (SecurityScheme)

**Published as:** `RAML 1.0 SecurityScheme`

**Used by EAPI only.** Defines `client_id` and `client_secret` headers and documents 401/403 responses.

```raml
securitySchemes:
  client-id-enforcement: !include
    exchange_modules/{groupId}/security-schemes-oms/1.0.1/security-schemes-oms.raml

/orders:
  securedBy: [client-id-enforcement]
```

### How to Publish Fragments to Exchange

```
1. Open Design Center → Create New Fragment
2. Choose fragment type:
   Library / Trait / ResourceType / SecurityScheme
3. Write the RAML content
4. Click Publish → fill in:
   Name    : common-data-types
   Version : 1.0.0
   Type    : Fragment
5. Click Publish to Exchange

6. In your API RAML → Exchange Dependencies
7. Search your fragment name
8. Click Add → it appears in exchange_modules/
```

---

## ⚙️ Mule Implementation

### Project Naming Convention

| Project | artifactId | Port |
|---|---|---|
| System API | `order-management-db-sapi-impl` | 8081 |
| Process API | `order-management-process-api-impl` | 8082 |
| Experience API | `order-management-exp-api-impl` | 8083 |

> Note: `artifactId` uses `-impl` suffix to distinguish Mule app from RAML spec in Exchange.

### How to Create Project from RAML

```
1. Anypoint Studio → File → New → Mule Project
2. Project Name: order-management-db-sapi
3. Check: "Specify API definition file location"
4. Select: Design Center
5. Search and select your published RAML
6. Finish → Studio auto-generates:
   - APIKit Router config
   - Skeleton flows for each endpoint
   - Main flow with HTTP Listener
```

### Flow Structure (all 3 APIs)

```
Main XML
  └── main flow
        ├── HTTP Listener
        ├── APIKit Router
        └── error-handler ref="global-error-handler"

  └── endpoint flow (APIKit generated)
        ├── logger (request received)
        ├── flow-ref → sub-flow
        └── logger (response sent)

Sub-flow files (business logic)
  ├── place-orders-flow.xml      or  stock-availability-flow.xml
  └── validate-order-flow.xml    (PAPI only)

Global files
  ├── global-configs.xml         ← all connector configs
  └── global-error-handler.xml   ← all error handlers
```

### Key Implementation Details

#### Atomic Stock Decrement (Race Condition Prevention)

```sql
UPDATE inventory
SET    available_quantity = available_quantity - :qty
WHERE  sku = :sku
AND    available_quantity >= :qty
```

If `rowsAffected = 0` → stock was unavailable → order not placed. This prevents two customers from ordering the last unit simultaneously.

#### orderId Generation (Process API)

```dataweave
%dw 2.0
output application/json
---
"ORD-" ++
(now() as String {format: "yyyyMMdd"}) ++
"-" ++
((randomInt(900000) + 100000) as String)
// Result: ORD-20261012-000091
```

#### orderId Flow Across Layers

```
EAPI  → receives request  → NO orderId (RAML rejects if present)
PAPI  → generates orderId → sends to SAPI
SAPI  → stores orderId    → returns success/failure
PAPI  → returns orderId   → in itemResults
EAPI  → returns to caller → customer sees orderId
```

#### Customer Upsert Strategy

```sql
INSERT INTO customers (...) VALUES (...)
ON DUPLICATE KEY UPDATE
  full_name     = VALUES(full_name),
  email         = VALUES(email),
  address_line1 = VALUES(address_line1),
  ...
```

`phoneNumber` is the unique key. Existing customers are updated automatically — no separate GET needed.

#### Stock Response — Edge Cases

| Scenario | `stockStatus` | `isAvailable` | `availableQuantity` |
|---|---|---|---|
| Stock > 0 | `IN_STOCK` | `true` | actual qty |
| Stock = 0 | `OUT_OF_STOCK` | `false` | 0 |
| SKU not in DB | `SKU_NOT_FOUND` | `false` | 0 |

---

## 🔐 Properties & Secure Configuration

### Properties File Structure

```
src/main/resources/properties/
├── dev.properties            ← actual values (gitignored)
└── dev.properties.template   ← placeholder (committed to git)
```

### Loading Properties in global-configs.xml

```xml
<configuration-properties
    file="properties/${mule.env}.properties"
    doc:name="Config Properties"/>

<secure-properties:config
    name="Secure_Properties_Config"
    file="properties/${mule.env}.properties"
    key="${mule.secure.key}"
    doc:name="Secure Properties Config"/>
```

### Encrypting the Database Password

The MySQL password is encrypted using the **MuleSoft Secure Properties Tool**:

```bash
# Download the tool from MuleSoft documentation
# Run encryption:
java -jar secure-properties-tool.jar \
  string encrypt AES CBC ABCD1234EFGH5678 "yourPlainPassword"

# Result: something like
# ![YQu3+DO8gYV5F9giEMyiAkUFkd9q5LKc7EVOmesT6L20778wEKsK6gOHyRXAk/Ps]

# Use in properties file:
database.password=![YQu3+DO8gYV5F9giEMyiAkUFkd9q5LKc7EVOmesT6L20778wEKsK6gOHyRXAk/Ps]
```

**In DB connector config:**
```xml
<db:my-sql-connection
    host="${database.host}"
    port="${database.port}"
    user="${database.user}"
    password="${secure::database.password}"   ← secure:: prefix
    database="${database.name}"/>
```

**Secure key is provided at deployment time** — never stored in properties files or committed to code.

---

## 🚨 Global Error Handling

Each API has its own `global-error-handler.xml` with handlers specific to that layer:

### Error Handler Comparison

| Error Type | SAPI | PAPI | EAPI | HTTP Status |
|---|---|---|---|---|
| `APIKIT:BAD_REQUEST` | ✅ | ✅ | ✅ | 400 |
| `VALIDATION:INVALID_VALUE` | ✅ | ✅ | ❌ | 400 |
| `VALIDATION:EMPTY_COLLECTION` | ✅ | ✅ | ❌ | 400 |
| `APIKIT:NOT_FOUND` | ✅ | ✅ | ✅ | 404 |
| `APIKIT:METHOD_NOT_ALLOWED` | ✅ | ✅ | ✅ | 405 |
| `APIKIT:NOT_ACCEPTABLE` | ✅ | ✅ | ✅ | 406 |
| `APIKIT:UNSUPPORTED_MEDIA_TYPE` | ✅ | ✅ | ✅ | 415 |
| `DB:CONNECTIVITY` | ✅ | ❌ | ❌ | 500 |
| `DB:QUERY_EXECUTION` | ✅ | ❌ | ❌ | 500 |
| `HTTP:CONNECTIVITY` | ❌ | ✅ | ✅ | 503 |
| `HTTP:TIMEOUT` | ❌ | ✅ | ✅ | 504 |
| `MULE:EXPRESSION` | ❌ | ✅ | ❌ | 500 |
| `ANY` | ✅ | ✅ | ✅ | 500 |

### Why Layer-Specific Handlers?

- SAPI uses DB connectors → handles `DB:*` errors
- PAPI/EAPI use HTTP connectors → handle `HTTP:*` errors
- EAPI is passthrough → no `VALIDATION:*` needed
- Client ID Enforcement (401/403) is handled by the **Gateway policy**, not the error handler

### Standard Error Response Format

```json
{
  "errorCode":     "BAD_REQUEST",
  "message":       "Invalid or missing input in request payload",
  "correlationId": "ab5afca0-3980-11f1-b1ab-5cfb3acc7915",
  "timestamp":     "2026-04-22T06:00:00Z"
}
```

### Logging Standards

```
✅ LOG:                        ❌ NEVER LOG:
correlationId                  phoneNumber (PII)
httpStatus code                email (PII)
sizeOf(items) — count only     fullName (PII)
endpoint entered/exited        address (PII)
rowsAffected count             SKU + quantity together
success: true/false            orderId
error type and description     DB column values
requestStatus outcome          client credentials
```

---

## 🚀 Local Setup & Running

### Prerequisites

- Anypoint Studio 7.x with Mule 4.6.x runtime
- Java 17
- MySQL Workbench (to verify DB)
- Postman

### Steps

**1. Clone the repository**
```bash
git clone https://github.com/YOUR_USERNAME/order-management-system.git
cd order-management-system
```

**2. Set up properties files**
```bash
# For each API project:
cp src/main/resources/properties/dev.properties.template \
   src/main/resources/properties/dev.properties

# Fill in your actual values in dev.properties
```

**3. Set runtime argument for secure key**

In Anypoint Studio:
```
Run → Run Configurations → Arguments tab
VM Arguments:
-Dmule.env=dev -Dmule.secure.key=ABCD1234EFGH5678
```

**4. Start APIs in order**

```
1. Start SAPI  → wait for "Started app 'order-management-db-sapi'"
2. Start PAPI  → wait for "Started app 'order-management-process-api'"
3. Start EAPI  → wait for "Started app 'order-management-exp-api'"
```

**5. Verify all running**
```
SAPI  → http://localhost:8081/system/stock/availability
PAPI  → http://localhost:8082/process/orders
EAPI  → http://localhost:8083/experience/orders
```

---

## 🧪 Postman Testing

### Create Environment

```
Postman → Environments → Add New

Name: OMS-Local

Variables:
┌─────────────────┬───────────────────────────────────────┐
│ Variable        │ Value                                  │
├─────────────────┼───────────────────────────────────────┤
│ base_url_sapi   │ http://localhost:8081/system           │
│ base_url_papi   │ http://localhost:8082/process          │
│ base_url_eapi   │ http://localhost:8083/experience       │
│ client_id       │ your-client-id-from-exchange           │
│ client_secret   │ your-client-secret-from-exchange       │
└─────────────────┴───────────────────────────────────────┘
```

### Test Collection Structure

```
📁 OMS — Big Billion Day
├── 📁 SAPI — Direct Tests
│   ├── ✅ Stock Check — All In Stock
│   ├── ✅ Stock Check — Mixed Status
│   ├── ✅ Stock Check — Unknown SKU (SKU_NOT_FOUND)
│   ├── ✅ Customer Upsert — New Customer
│   ├── ✅ Customer Upsert — Existing Customer (Update)
│   ├── ✅ Place Order — Stock Available
│   ├── ✅ Place Order — Out of Stock
│   └── ❌ Stock Check — Empty SKU Array (400)
│
├── 📁 PAPI — Business Logic Tests
│   ├── ✅ Place Orders — SUCCESS (all placed)
│   ├── ✅ Place Orders — PARTIAL_SUCCESS
│   ├── ✅ Place Orders — FAILED (all out of stock)
│   ├── ❌ Validation — orderId present (400)
│   ├── ❌ Validation — empty items (400)
│   ├── ❌ Validation — blank SKU (400)
│   ├── ❌ Validation — invalid phone (400)
│   ├── ❌ Validation — invalid email (400)
│   └── ✅ Stock Availability — Mix of statuses
│
└── 📁 EAPI — End-to-End Tests
    ├── ✅ Full E2E — SUCCESS
    ├── ✅ Full E2E — PARTIAL_SUCCESS
    ├── ✅ Stock Check — via EAPI
    ├── ❌ Missing client_id header (401)
    ├── ❌ Wrong credentials (401)
    ├── ❌ Wrong HTTP method — GET (405)
    └── ❌ Wrong Content-Type — text/plain (415)
```

### Key Test Requests

**Place Orders Request:**
```json
POST {{base_url_eapi}}/orders
Headers:
  client_id:     {{client_id}}
  client_secret: {{client_secret}}
  Content-Type:  application/json

Body:
{
  "customer": {
    "fullName":    "Ravi Kumar",
    "phoneNumber": "9876543210",
    "email":       "ravi.kumar@example.com",
    "address": {
      "line1":      "Flat 1204, Lakeview Residency",
      "line2":      "Near City Mall",
      "city":       "Bengaluru",
      "state":      "KA",
      "postalCode": "560102"
    }
  },
  "orderChannel":       "MOBILE_APP",
  "paymentMode":        "UPI",
  "deliveryPreference": "EXPRESS",
  "items": [
    { "sku": "MOB-IPH15-128-BLK", "quantity": 1 },
    { "sku": "LAP-DELL-5480",     "quantity": 2 },
    { "sku": "EAR-BT-SONY-WH",   "quantity": 1 },
    { "sku": "TV-SAM-55-UHD",     "quantity": 1 },
    { "sku": "TAB-SAM-S9",        "quantity": 3 }
  ]
}
```

**Stock Availability Request:**
```json
POST {{base_url_eapi}}/stock/availability
Headers:
  client_id:     {{client_id}}
  client_secret: {{client_secret}}
  Content-Type:  application/json

Body:
{
  "skus": [
    "MOB-IPH15-128-BLK",
    "LAP-DELL-5480",
    "EAR-BT-SONY-WH",
    "TV-SAM-55-UHD",
    "INVALID-SKU-999"
  ]
}
```

---

## ☁️ CloudHub Deployment

### Deployment Order

```
Always deploy bottom → up:
  1. Deploy SAPI first
  2. Deploy PAPI after SAPI is running
  3. Deploy EAPI last
```

### Step-by-Step Deployment

**1. Update properties for CloudHub**

```properties
# CloudHub apps communicate on port 80
# Update PAPI dev.properties:
system.api.host=order-management-db-sapi-dev.cloudhub.io
system.api.port=80
system.api.basePath=system

# Update EAPI dev.properties:
process.api.host=order-management-process-api-dev.cloudhub.io
process.api.port=80
process.api.basePath=process
```

**2. Deploy from Anypoint Studio**

```
Right-click project
→ Anypoint Platform → Deploy to CloudHub 2.0
→ Fill in:
  App Name    : order-management-db-sapi-dev
  Runtime     : 4.6.x
  Worker Size : 0.1 vCore
  Region      : US East

→ Properties tab → Add:
  mule.env              = dev
  mule.secure.key       = ABCD1234EFGH5678
  (for EAPI also add:)
  anypoint.platform.client_id     = your-platform-client-id
  anypoint.platform.client_secret = your-platform-client-secret
  api.id                          = your-api-manager-instance-id

→ Deploy
```

**3. Verify deployment**

```
Runtime Manager → your app
Status should show: ✅ STARTED

CloudHub URLs:
SAPI  → https://order-management-db-sapi-dev.cloudhub.io/system/stock/availability
PAPI  → https://order-management-process-api-dev.cloudhub.io/process/orders
EAPI  → https://order-management-exp-api-dev.cloudhub.io/experience/orders
```

> **Note:** CloudHub 2.0 uses HTTPS by default on port 443.

---

## 🔒 Applying Policies — Client ID Enforcement

Client ID Enforcement is applied **only on the Experience API**. PAPI and SAPI are internal and not exposed publicly.

### Step 1 — Create API Instance in API Manager

```
Anypoint Platform → API Manager
→ Add API → Add new API
→ Runtime     : Mule Gateway
→ API name    : order-management-exp-api
→ Version     : v1
→ Save → Note the API Instance ID
```

### Step 2 — Add Autodiscovery to EAPI

```xml
<!-- global-configs.xml -->
<api-gateway:autodiscovery
    apiId="${api.id}"
    flowRef="order-management-exp-api-main"
    doc:name="API Autodiscovery"/>
```

```properties
# dev.properties
api.id=YOUR_API_INSTANCE_ID
```

Provide these as CloudHub deployment properties:
```
anypoint.platform.client_id     = your-business-group-client-id
anypoint.platform.client_secret = your-business-group-client-secret
api.id                          = your-api-instance-id
```

### Step 3 — Apply Client ID Enforcement Policy

```
API Manager → your API instance
→ Policies → Add Policy
→ Search: Client ID Enforcement
→ Configure:
  Credentials Origin  : HTTP Headers
  Client ID Header    : client_id
  Client Secret Header: client_secret
→ Apply
```

### Step 4 — Get Consumer Credentials

```
Anypoint Exchange → find your EAPI
→ Request Access → Create new application
  App Name: postman-test-app
→ Request Access → Approve
→ View credentials:
  client_id     : xxxxxxxxxxxxxxxxxx
  client_secret : xxxxxxxxxxxxxxxxxx
```

### How the Policy Works

```
Request arrives at CloudHub
        │
        ▼
Anypoint Gateway intercepts
Validates client_id + client_secret headers
        │
        ├── Missing or invalid → 401 (Gateway response)
        ├── App not approved  → 403 (Gateway response)
        │
        └── Valid ────────────────────────────────────────►
                                                    Mule flow starts
                                                    Request processed
                                                    Response returned
```

> **Note:** 401/403 responses are returned directly by the Gateway in its own format. They do not pass through the Mule error handler. This is expected behavior for policy-level enforcement.

---

## 📝 Assumptions & Limitations

### Assumptions

| # | Assumption |
|---|---|
| 1 | `phoneNumber` is the unique customer identifier across all requests |
| 2 | `orderId` must NOT be present in the incoming request — it is generated internally by the Process API |
| 3 | Stock decrement is immediate and atomic at the database level using `WHERE available_quantity >= qty` |
| 4 | Unknown SKUs (not in inventory table) return `stockStatus: SKU_NOT_FOUND` without throwing an error |
| 5 | Client ID Enforcement policy is applied only on the Experience API |
| 6 | PAPI and SAPI are not exposed publicly — protected by network isolation |
| 7 | `orderChannel` accepts `MOBILE_APP` and `WEB_APP` values |
| 8 | All APIs communicate using `application/json` media type |

### Limitations

| # | Limitation |
|---|---|
| 1 | **No distributed transaction rollback** — if PAPI fails mid-loop after placing some orders, partial DB state is possible |
| 2 | **401/403 error format** is governed by the Anypoint Gateway Client ID Enforcement policy and does not match the standard `errorCode/message/correlationId/timestamp` structure |
| 3 | **filess.io free tier** has connection limits and may experience latency on the shared MySQL server |
| 4 | **CloudHub 0.1 vCore** — deployed on minimum worker size; not suitable for production load |
| 5 | **randomInt orderId** — uses `randomInt(900000) + 100000` for the numeric suffix; collision possible under extreme load (use UUID or DB sequence in production) |
| 6 | **No retry mechanism** — if System API is temporarily unavailable, the request fails immediately without retry |

---

## 📁 Repository Structure

```
order-management-system/
│
├── README.md                                    ← This file
├── database/
│   └── output_file.sql                          ← Full DB schema + data
│
├── order-management-db-sapi/                    ← System API
│   ├── src/main/mule/
│   │   ├── order-management-db-sapi.xml
│   │   ├── flows/
│   │   │   ├── stock-availability.xml
│   │   │   ├── customer-upsert.xml
│   │   │   └── order-place.xml
│   │   └── global/
│   │       ├── global-configs.xml
│   │       └── global-error-handler.xml
│   └── src/main/resources/
│       ├── api/                                 ← RAML files
│       └── properties/
│           └── dev.properties.template
│
├── order-management-process-api/                ← Process API
│   ├── src/main/mule/
│   │   ├── order-management-process-api.xml
│   │   ├── flows/
│   │   │   ├── place-orders-flow.xml
│   │   │   └── stock-availability-flow.xml
│   │   └── global/
│   │       ├── global-configs.xml
│   │       └── global-error-handler.xml
│   └── src/main/resources/
│       ├── api/
│       └── properties/
│           └── dev.properties.template
│
└── order-management-exp-api/                    ← Experience API
    ├── src/main/mule/
    │   ├── order-management-exp-api.xml
    │   ├── flows/
    │   │   ├── place-orders-flow.xml
    │   │   └── stock-availability-flow.xml
    │   └── global/
    │       ├── global-configs.xml
    │       └── global-error-handler.xml
    └── src/main/resources/
        ├── api/
        └── properties/
            └── dev.properties.template
```

---

## 👤 Author

**Intern Assignment — MuleSoft Order Management System**
Built as part of MuleSoft Developer Internship Program

---

*Built with ❤️ using MuleSoft Anypoint Platform*
